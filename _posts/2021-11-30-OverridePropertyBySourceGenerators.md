---
layout: post
title:  "使用Source Generators修改属性的Get和Set的值"
date:   2021-11-30 21:42:55 +0800--
categories: [.NET 5]
tags: [.NET Core, Source Generators, 属性,Get,Set]  
---

### 前言
博主最近遇到了一个需求, 需要对返回实体中的部分字段数据做脱敏处理, 对于这个需求,基本上有三个方案可以实现:
- 在每个需要脱敏属性的Set方法中调用脱敏方法, 该方法的缺点是在很多地方都会出现脱敏方法的调用, 也会让属性的实现变的复杂, 所以一开始就被Pass了。
- 通过AOP的方式实现
- 通过Source Generators的方式实现

### 方案一、每个Set中调用脱敏方法

```CSharp
public class ExampleModel
{
    private string _fullName;

    public string FullName 
    { 
        get
        {
            return _fullName;
        }
        set
        {
            _fullName = 脱敏方法(value)
        }
    }
}
```
由上述示例代码可以看到, 整体的代码相对冗余, 不够简洁。

### 方案二、通过AOP的方式实现

```CSharp
public class ExampleModel
{
    [脱敏Attribute]
    public string FullName { get; set; }
}
```

```CSharp
public class 脱敏Attribute : Attribute
{
    public void OnSetValue(object value)
    {    
       value = 脱敏方法(value);
    }
}
```
**注意: 以上内容只是类似于伪代码的实现**

通过对比方案二和方案一, 可以发现方案二的实现比方案一简洁的多, 同时使用上也方便的多, 对于需脱敏的属性, 只要增加特性即可。

这个方案的缺点是, .Net框架中本身并没有自带AOP框架的实现,需要引入第三方的AOP框架。

### 方案三、通过Source Generators的方式实现

#### 1. 实体类
```CSharp
public partial class ExampleModel
{
    [脱敏Attribute]
    private string _fullName;
}
```
**注意1: 实体类加上了partial修饰符,这是一个部分类**  
**注意2: 实体类中包含的是一个私有属性**

#### 2. Source Generators类：
Source Generators的使用方式，请参照：[Source Generators 探索](https://jerryluo.com/2021/10/30/SourceGeneratorsExplore/)
```CSharp
namespace SourceGeneratorSamples
{
    [Generator]
    public class DesensitizationGenerator : ISourceGenerator
    {
        /// <summary>
        /// 特性类代码
        /// </summary>
        private const string attributeText = @"
        using System;
        namespace Desensitization
        {
            [AttributeUsage(AttributeTargets.Field, Inherited = false, AllowMultiple = false)]
            sealed class DesensitizationAttribute : Attribute
            {
                public DesensitizationAttribute()
                {
                }
            }
        }
        ";

        public void Initialize(GeneratorInitializationContext context)
        {
            //语法树
            context.RegisterForSyntaxNotifications(() => new SyntaxReceiver());
        }

        public void Execute(GeneratorExecutionContext context)
        {
            //生成特性类代码
            context.AddSource("DesensitizationAttribute", attributeText);

            if (!(context.SyntaxReceiver is SyntaxReceiver receiver))
                return;

            CSharpParseOptions options = (context.Compilation as CSharpCompilation).SyntaxTrees[0].Options as CSharpParseOptions;
            Compilation compilation = context.Compilation.AddSyntaxTrees(CSharpSyntaxTree.ParseText(SourceText.From(attributeText, Encoding.UTF8), options));
            INamedTypeSymbol attributeSymbol = compilation.GetTypeByMetadataName("Desensitization.DesensitizationAttribute");

            //循环所有属性
            List<IFieldSymbol> fieldSymbols = new List<IFieldSymbol>();
            foreach (FieldDeclarationSyntax field in receiver.CandidateFields)
            {
                SemanticModel model = compilation.GetSemanticModel(field.SyntaxTree);
                foreach (VariableDeclaratorSyntax variable in field.Declaration.Variables)
                {
                    IFieldSymbol fieldSymbol = model.GetDeclaredSymbol(variable) as IFieldSymbol;
                    //获取包含了指定特性的属性
                    if (fieldSymbol.GetAttributes().Any(ad => ad.AttributeClass.Equals(attributeSymbol, SymbolEqualityComparer.Default)))
                    {
                        fieldSymbols.Add(fieldSymbol);
                    }
                }
            }

            //根据包含属性的类, 生成对应的部分类
            foreach (IGrouping<INamedTypeSymbol, IFieldSymbol> group in fieldSymbols.GroupBy(f => f.ContainingType))
            {
                string classSource = ProcessClass(group.Key, group.ToList(), attributeSymbol, context);
                context.AddSource($"{group.Key.Name}_desensitization.cs", classSource);
            }
        }

        /// <summary>
        /// 生成实体类代码
        /// </summary>
        /// <param name="classSymbol"></param>
        /// <param name="fields"></param>
        /// <param name="attributeSymbol"></param>
        /// <param name="context"></param>
        /// <returns></returns>
        private string ProcessClass(INamedTypeSymbol classSymbol, List<IFieldSymbol> fields, ISymbol attributeSymbol, GeneratorExecutionContext context)
        {
            if (!classSymbol.ContainingSymbol.Equals(classSymbol.ContainingNamespace, SymbolEqualityComparer.Default))
            {
                return null; //TODO: issue a diagnostic that it must be top level
            }

            string namespaceName = classSymbol.ContainingNamespace.ToDisplayString();

            //开始生成部分类
            StringBuilder source = new StringBuilder($@"
                namespace {namespaceName}
                {{
                    public partial class {classSymbol.Name}
                    {{
                ");

            //生成部分类的每一个字段
            foreach (IFieldSymbol fieldSymbol in fields)
            {
                ProcessField(source, fieldSymbol, attributeSymbol);
            }

            source.Append("} }");
            return source.ToString();
        }

        /// <summary>
        /// 生成属性代码
        /// </summary>
        /// <param name="source"></param>
        /// <param name="fieldSymbol"></param>
        /// <param name="attributeSymbol"></param>
        private void ProcessField(StringBuilder source, IFieldSymbol fieldSymbol, ISymbol attributeSymbol)
        {
            string fieldName = fieldSymbol.Name;
            ITypeSymbol fieldType = fieldSymbol.Type;

            // get the AutoNotify attribute from the field, and any associated data
            AttributeData attributeData = fieldSymbol.GetAttributes().Single(ad => ad.AttributeClass.Equals(attributeSymbol, SymbolEqualityComparer.Default));
            TypedConstant overridenNameOpt = attributeData.NamedArguments.SingleOrDefault(kvp => kvp.Key == "PropertyName").Value;

            string propertyName = chooseName(fieldName, overridenNameOpt);
            if (propertyName.Length == 0 || propertyName == fieldName)
            {
                return;
            }

            source.Append($@"
                public {fieldType} {propertyName} 
                {{
                    get 
                    {{
                        return this.{fieldName};
                    }}

                    set
                    {{
                        this.{fieldName} = 脱敏方法(value);
                    }}
                }}
            ");

            //根据私有属性名(如:_fullName),转换为公有属性名(如:FullName)
            string chooseName(string fieldName, TypedConstant overridenNameOpt)
            {
                if (!overridenNameOpt.IsNull)
                {
                    return overridenNameOpt.Value.ToString();
                }

                fieldName = fieldName.TrimStart('_');
                if (fieldName.Length == 0)
                    return string.Empty;

                if (fieldName.Length == 1)
                    return fieldName.ToUpper();

                return fieldName.Substring(0, 1).ToUpper() + fieldName.Substring(1);
            }

        }

        /// <summary>
        /// Created on demand before each generation pass
        /// </summary>
        class SyntaxReceiver : ISyntaxReceiver
        {
            public List<FieldDeclarationSyntax> CandidateFields { get; } = new List<FieldDeclarationSyntax>();

            /// <summary>
            /// Called for every syntax node in the compilation, we can inspect the nodes and save any information useful for generation
            /// </summary>
            public void OnVisitSyntaxNode(SyntaxNode syntaxNode)
            {
                // any field with at least one attribute is a candidate for property generation
                if (syntaxNode is FieldDeclarationSyntax fieldDeclarationSyntax
                    && fieldDeclarationSyntax.AttributeLists.Count > 0)
                {
                    CandidateFields.Add(fieldDeclarationSyntax);
                }
            }
        }
    }
}

```

#### 3. 生成的代码1-脱敏的特性类
```CSharp
using System;

namespace Desensitization
{
    [AttributeUsage(AttributeTargets.Field, Inherited = false, AllowMultiple = false)]
    sealed class DesensitizationAttribute : Attribute
    {
        public DesensitizationAttribute()
        {
        }
    }
}
```

#### 4. 生成的代码2-部分实体类
```CSharp
public partial class ExampleViewModel
{
    public string FullName
    {
        get
        {
            return this._fullName;
        }

        set
        {
            this._fullName = 脱敏方法(value);
        }
    }
}
```

结合刚刚我们自定义的部分实体类, 我们可以发现实际上方法三的最终代码和方法一是一致的,只不过我们通过SourceGenerator实现了对冗余代码的生成, 而且这部分冗余代码在我们实际开发过程中是不可见的, 是在编译的过程中,由编译器生成代码并将这部分代码打包到DLL文件中，这样做的好处是：
- 代码和**方案二**一样简洁
- 无须引入第三方的框架
- 性能相对会比**方案二**更好