---
layout: post
title:  "使用Azure DevOps API批量导入工作项"
date:   2022-12-25 21:42:55 +0800--
categories: [Azure DevOps]
tags: [Azure DevOps, Azure DevOps APIs, WorkItems]  
---

### 1. 前言

Azure DevOps可以通过Excel文件导入工作项, 同时有可以通过[Azure DevOps API](https://docs.microsoft.com/en-us/rest/api/azure/devops)动态的导入。

### 2. 如何实现

首先， 准备好一个文件， 里面录入工作项数据。
![工作项数据](https://devblogs.microsoft.com/premier-developer/wp-content/uploads/sites/31/2022/05/word-image-4.png)

示例代码:

```C#
string azureDevOpsOrganizationUrl = "https://dev.azure.com/{Organization}/{Project}/";
private void ImportWorkItems()
{
    string[] workItems = File.ReadAllLines("<Path to CSV file>");

    var personalAccessToken = "<Azure DevOps PAT token>";

    string credentials = Convert.ToBase64String(System.Text.ASCIIEncoding.ASCII.GetBytes(string.Format("{0}:{1}", "", personalAccessToken)));
    
    //Just skipping the CSV file header
    foreach (var row in workItems.Skip(1))
    {
        var columns = row.Split(',');

        var type = columns[0];
        var title = columns[1];
        var description = columns[2];

        using (var client = new HttpClient())
        {
            client.BaseAddress = new Uri(azureDevOpsOrganizationUrl);
            client.DefaultRequestHeaders.Accept.Clear();
            client.DefaultRequestHeaders.Accept.Add(new MediaTypeWithQualityHeaderValue("application/json-patch+json"));
            client.DefaultRequestHeaders.Add("User-Agent", "ManagedClientConsoleAppSample");
            client.DefaultRequestHeaders.Authorization = new AuthenticationHeaderValue("Basic", credentials);

            //connect to the REST endpoint
            string uri = String.Format("_apis/wit/workitems/${0}?bypassRules=true&api-version=6.0", type);
            //building JSON request body
            var jsonstr = "[{\"op\": \"add\", \"path\": \"/fields/System.Title\",\"value\": \"" + title + "\" }";
            jsonstr += ",{\"op\": \"add\", \"path\": \"/fields/System.Description\",\"value\": \"" + description + "\" }";
            jsonstr += "]"; 
            HttpContent body = new StringContent(jsonstr, Encoding.UTF8, "application/json-patch+json");
            HttpResponseMessage response = client.PostAsync(uri, body).Result;                    
            // check to see if we have a successful respond
            if (response.IsSuccessStatusCode)
            {
                ...
            }
            else
            {
                ...
            }
        }
    }
}
```

代码执行结果为:
![执行结果](https://devblogs.microsoft.com/premier-developer/wp-content/uploads/sites/31/2022/05/graphical-user-interface-text-application-email-5.png)

### 3. 参考资料
[Work Items – Create – REST API](https://learn.microsoft.com/en-us/rest/api/azure/devops/wit/work-items/create?view=azure-devops-rest-6.0&tabs=HTTP)

[Get started with the REST APIs](https://learn.microsoft.com/en-us/azure/devops/integrate/how-to/call-rest-api?view=azure-devops)