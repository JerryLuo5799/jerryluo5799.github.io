---
layout: post
title:  ".NET9详解系列之二: C#13的新特性 "
date:   2024-11-20 21:42:55 +0800--
categories: [.NET]
tags: [.NET9]  
---

## 前言

C# 13作为.NET 9的重要组成部分，带来了许多令人兴奋的新特性和改进。这些新特性不仅提升了代码的灵活性和可读性，还在性能和开发体验上有了显著的提升。对于初级开发人员来说，了解这些新特性将有助于更快地掌握C#语言的精髓，并写出更高效、更优雅的代码。本文将详细介绍C# 13的新特性，并通过具体的示例代码帮助大家更好地理解和应用这些特性。

## 一、扩展类型（Extension Types）

### 1.1 传统扩展方法的局限性及旧方法实现

在C#的早期版本中，我们可以通过扩展方法为现有类型添加新的功能。然而，扩展方法只能是静态方法，并且只能添加方法，不能添加属性或其他成员。这在一定程度上限制了我们对现有类型的扩展能力。

```csharp
using System;

var zhangsan = new Person();
Console.WriteLine(zhangsan.GetAge());

public class Person
{
    public string Name { get; set; }
    public DateTime Birthday { get; set; }
}

public static class PersonExtension
{
    public static int GetAge(this Person person) => DateTime.Now.Year - person.Birthday.Year;
}
```

在上面的示例中，我们通过扩展方法为`Person`类添加了`GetAge`方法。这种方式虽然可以扩展方法，但无法扩展属性，且语法相对繁琐。

### 1.2 扩展类型的引入及新方法实现

C# 13引入了扩展类型（Extension Types），允许我们向现有类添加新的方法、属性，甚至是静态成员，而无需修改原始类的代码。这极大地增强了代码的灵活性和可维护性。

#### 1.2.1 隐式扩展（implicit extension）

隐式扩展允许我们为现有类型添加新的实例成员，包括方法和属性。通过使用`implicit extension`关键字，我们可以定义一个扩展类型，该类型中的成员可以直接在原始类型上使用。

```csharp
public implicit extension PersonExtension for Person
{
    // 扩展实例方法
    public int GetAge()
    {
        return DateTime.Today.Year - this.DateOfBirth.Year;
    }

    // 扩展实例属性
    public int Age => DateTime.Today.Year - this.DateOfBirth.Year;
}
```

在上面的示例中，我们为`Person`类添加了`GetAge`方法和`Age`属性。通过`this`关键字，我们可以访问当前实例对象的成员。

```csharp
var person = new Person { DateOfBirth = new DateTime(2000, 1, 1) };
Console.WriteLine(person.Age); // 输出当前年份减去2000年的结果
```

#### 1.2.2 显式扩展（explicit extension）

显式扩展则允许我们在需要的时候进行多态转换，进一步增强了代码的灵活性。例如，我们可以为一个基类定义一个显式扩展类型，然后在需要的时候将其转换为扩展类型，以访问扩展类型中的成员。

```csharp
public explicit extension Male for Person
{
    public string LikeSport => "football";
}
```

在使用时，我们可以将`Person`对象显式转换为`Male`类型，然后访问`LikeSport`属性。

```csharp
var person = new Person();
if (person is Male male)
{
    Console.WriteLine(male.LikeSport); // 输出"football"
}
```

## 二、params集合增强

### 2.1 传统params的局限性及旧方法实现

在C#的早期版本中，`params`关键字只能用于数组类型，这在某些场景下可能不够灵活，尤其是在处理大量参数时。

```csharp
private static int Sum(params int[] values)
{
    int sum = 0;
    foreach (var item in values)
    {
        sum += item;
    }
    return sum;
}
```

在上面的示例中，我们使用`params int[]`作为参数类型，这样在调用`Sum`方法时，可以传递任意数量的整数参数。

### 2.2 params集合增强及新方法实现

C# 13扩展了`params`关键字的使用范围，使其不仅限于数组，还可以应用于`System.Span<T>`、`System.ReadOnlySpan<T>`以及实现了`System.Collections.Generic.IEnumerable<T>`的类型。这使得在处理大量参数时更加高效和灵活。

```csharp
private static int Sum(params ReadOnlySpan<int> values)
{
    int sum = 0;
    foreach (var item in values)
    {
        sum += item;
    }
    return sum;
}
```

在上面的示例中，我们使用`params ReadOnlySpan<int>`作为参数类型，这样在调用`Sum`方法时，可以传递任意数量的整数参数，而无需显式创建数组。

## 三、锁对象（Lock）

### 3.1 传统锁的不足及旧方法实现

在多线程编程中，传统的`lock`语句需要一个对象作为锁的标识，这在某些情况下可能会导致锁的误用，从而引发线程安全问题。

```csharp
object lockObj = new object();
lock (lockObj)
{
    // 在这里执行需要线程安全的操作
}
```

在上面的示例中，我们使用`lock`语句和一个对象作为锁的标识，来确保线程安全。

### 3.2 新的锁对象及新方法实现

C# 13引入了`System.Threading.Lock`类型，提供了改进的线程同步机制。通过`Lock.EnterScope()`方法，可以进入一个独占作用域，从而更安全地管理线程同步。

```csharp
var lockObj = new Lock();
using (lockObj.EnterScope())
{
    // 在这里执行需要线程安全的操作
}
```

在上面的示例中，我们创建了一个`Lock`对象，并通过`EnterScope()`方法进入一个独占作用域。在`using`块中，我们可以安全地执行需要线程同步的操作。

## 四、索引器改进

### 4.1 传统索引器的局限性及旧方法实现

在C#的早期版本中，索引器只能从头开始计数，这在某些场景下可能不够灵活，尤其是在需要从末尾开始访问元素时。

```csharp
public class IndexedData
{
    public string[] Items { get; set; } = new string[5];
}

var data = new IndexedData();
data.Items[2] = "Second";
data.Items[3] = "Third";
```

在上面的示例中，我们通过传统的索引器初始化数组元素。

### 4.2 尾部索引（^符号）及新方法实现

C# 13引入了尾部索引的概念，允许我们从集合的末尾开始计数。通过使用`^`符号，我们可以更直观地访问集合末尾的元素。

```csharp
var data = new IndexedData
{
    Items = { [2] = "Second", [3] = "Third" },
    [^1] = "First", // 从末尾开始的第一个元素
    [^2] = "Fourth" // 从末尾开始的第二个元素
};
```

在上面的示例中，我们使用`[^1]`和`[^2]`来初始化数组的末尾元素。这样，在访问这些元素时，我们可以更直观地理解它们的位置。

## 五、转义序列`\e`

### 5.1 传统转义序列的不足及旧方法实现

在C#的早期版本中，如果我们需要表示ESC字符，通常需要使用`\u001b`或`\x1b`这样的十六进制转义序列，这在代码中不够直观。

```csharp
Console.WriteLine("\u001b[31mHello, World!\u001b[0m"); // 输出红色的"Hello, World!"
```

在上面的示例中，我们使用`\u001b`来表示ESC字符，从而控制控制台输出的颜色。

### 5.2 新的转义序列`\e`及新方法实现

C# 13引入了`\e`作为ESC字符的转义序列，这使得在代码中表示ESC字符更加直观和简洁。

```csharp
Console.WriteLine("\e[31mHello, World!\e[0m"); // 输出红色的"Hello, World!"
```

在上面的示例中，我们使用`\e`来表示ESC字符，从而更直观地控制控制台输出的颜色。

## 六、部分属性（Partial Properties）

### 6.1 传统属性的局限性及旧方法实现

在C#的早期版本中，属性的定义和实现必须在同一个文件中完成，这在某些场景下可能不够灵活，尤其是在需要将属性的定义和实现分离时。

```csharp
public partial class Person
{
    private string _name;
    public string Name
    {
        get => _name;
        set => _name = value?.Trim();
    }
}
```

在上面的示例中，我们在同一个文件中定义并实现了`Name`属性。

### 6.2 部分属性的引入及新方法实现

C# 13允许我们将属性的定义和实现分布在不同的文件中，这提高了代码的组织性和可维护性。

```csharp
public partial class Person
{
    public partial string Name { get; set; }
}

public partial class Person
{
    private string _name;
    public partial string Name
    {
        get => _name;
        set => _name = value?.Trim();
    }
}
```

在上面的示例中，我们通过`partial`关键字将`Name`属性的定义和实现分布在两个不同的文件中。这样，在需要修改属性的实现时，我们可以只修改其中一个文件，而不会影响其他文件。

## 七、方法组自然类型改进

### 7.1 传统方法组的不足及旧方法实现

在C#的早期版本中，当编译器遇到涉及方法组的表达式时，它会构造一个完整的候选方法集，这在处理大量候选方法时可能会导致性能问题。

```csharp
Action action = Method;
```

在上面的示例中，编译器会构造一个完整的候选方法集来确定`Method`的自然类型。

### 7.2 方法组自然类型改进及新方法实现

C# 13改进了编译器在处理方法组时的效率和准确性。通过在每个作用域内逐步削减候选方法集，移除那些明显不适用的候选方法，从而提高了编译时的效率。

```csharp
Action action = Method; // 编译器会更高效地确定Method的自然类型
```

在上面的示例中，编译器会更高效地确定`Method`的自然类型，并将其赋值给`action`变量。

## 八、在异步方法和迭代器中使用`ref`和`unsafe`

### 8.1 传统限制及旧方法实现

在C#的早期版本中，`ref`和`unsafe`代码不能在异步方法和迭代器中使用，这在某些需要高性能的场景下可能会限制我们的选择。

```csharp
public void Process(ref int value)
{
    // 在这里可以处理value
}
```

在上面的示例中，我们在普通方法中使用了`ref`局部变量`value`。

### 8.2 新的特性及新方法实现

C# 13允许我们在异步方法和迭代器中使用`ref`局部变量和`unsafe`上下文，从而在更多情况下使用这些特性。

```csharp
public async Task MethodAsync()
{
    ref int value = ref GetValue();
    // 在这里可以使用value进行操作
}

public unsafe IEnumerable<int> GetValues()
{
    fixed (int* ptr = &_value)
    {
        // 在这里可以使用ptr进行操作
        yield return *ptr;
    }
}
```

在上面的示例中，我们在异步方法`MethodAsync`中使用了`ref`局部变量`value`，并在迭代器`GetValues`中使用了`unsafe`上下文。

## 九、允许在泛型中将`ref struct`类型作为类型参数的参数

### 9.1 传统限制及旧方法实现

在C#的早期版本中，`ref struct`类型不能作为泛型类型参数的参数，这在某些需要高性能的场景下可能会限制我们的选择。

```csharp
public void Process<T>(T value) where T : struct
{
    // 在这里可以处理value
}
```

在上面的示例中，我们定义了一个泛型方法`Process`，它接受一个`struct`类型的参数。

### 9.2 新的特性及新方法实现

C# 13允许我们将`ref struct`类型作为泛型类型参数的参数，从而在泛型方法中使用`ref struct`类型。

```csharp
public void Process<T>(ref T value) where T : struct
{
    // 在这里可以处理value
}

ref int number = ref GetNumber();
Process(ref number);
```

在上面的示例中，我们定义了一个泛型方法`Process`，它接受一个`ref struct`类型的参数。然后，我们通过`ref`关键字获取了一个`int`类型的值，并将其传递给`Process`方法。

## 十、`field`关键字

### 10.1 传统属性的局限性及旧方法实现

在C#的早期版本中，当我们需要在属性的获取或设置过程中添加自定义逻辑时，必须手动声明一个私有字段作为该属性的后备存储。这种方式虽然灵活，但会导致代码冗长，尤其是在需要定义大量属性时，增加了开发的工作量和出错的风险。

```csharp
public class Person
{
    private string _name;
    public string Name
    {
        get => _name;
        set => _name = value?.Trim() ?? string.Empty;
    }
}
```

在上面的示例中，我们手动声明了一个私有字段`_name`作为`Name`属性的后备存储，并在`set`访问器中添加了自定义逻辑。

### 10.2 `field`关键字的引入及新方法实现

C# 13引入了`field`关键字，允许我们在属性访问器中直接引用编译器自动生成的后备字段，而无需显式声明该字段。这使得在实现具有自定义逻辑的属性时，代码更加简洁和直观。

```csharp
public class Person
{
    public string Name
    {
        get => field;
        set => field = value?.Trim() ?? string.Empty;
    }
}
```

在上面的代码中，`Name`属性的`get`访问器直接返回`field`，而`set`访问器在设置值时对输入进行了`Trim()`处理，并确保不会将`null`值赋给`field`。这里无需手动声明一个私有字段来存储`Name`的值，编译器会自动生成一个后备字段。

### 10.3 注意事项

- **上下文限制**：`field`关键字仅在属性的`get`或`set`访问器中有效，不能在其他地方使用。
- **字段冲突**：如果类中已经存在一个名为`field`的字段，那么在属性访问器中使用`field`关键字时，该字段会被隐藏。如果需要访问该字段，可以使用`@field`来显式引用它。
- **预览特性**：`field`关键字目前是C# 13中的预览特性，需要在项目文件中将语言版本设置为`preview`才能使用。

### 10.4 总结

`field`关键字的引入，使得在C#中实现具有自定义逻辑的属性变得更加简洁和高效。它减少了手动声明后备字段的需要，降低了代码的冗余度，同时保持了代码的可读性和灵活性。对于初级开发人员来说，这一特性使得属性的实现更加直观，有助于更快地掌握C#语言中属性的相关知识。

## 总结

C# 13带来的这些新特性，不仅提升了代码的灵活性和可读性，还在性能和开发体验上有了显著的提升。对于初级开发人员来说，掌握这些新特性将有助于更快地掌握C#语言的精髓，并写出更高效、更优雅的代码。在实际项目中，我们可以根据具体需求选择性地使用这些新特性，以提高代码的质量和可维护性。希望本文的介绍和示例能够帮助大家更好地理解和应用C# 13的新特性。

更多信息: [C#13 文档](https://learn.microsoft.com/zh-cn/dotnet/csharp/whats-new/csharp-13?wt.mc_id=MVP_324329)