**大局观**

Ocelot 旨在为使用 .NET 构建微服务/面向服务架构的用户提供一个统一的系统入口点。不过，Ocelot 也适用于任何使用 HTTP(S) 通信的系统，并可以运行在任何 ASP.NET Core 支持的平台上。

特别是我们希望能够轻松集成 IdentityServer 的引用令牌和持有者令牌（Bearer Tokens）。在当前的工作环境中，我们无法实现这一点，除非编写自己的 JavaScript 中间件来处理 IdentityServer 引用令牌。我们更倾向于使用现有的 IdentityServer 代码来完成此任务。

Ocelot 是一组按特定顺序排列的中间件。

Ocelot 会根据其配置修改 HttpRequest 对象的状态，直到它到达请求生成中间件。在这里，它会创建一个 HttpRequestMessage 对象，用于向下游服务发送请求。发出请求的中间件是 Ocelot 管道中的最后一个中间件，它不会调用下一个中间件。当下游服务的响应返回时，请求沿着 Ocelot 管道向上返回。某个中间件将 HttpResponseMessage 映射到 HttpResponse 对象，最终返回给客户端。这就是 Ocelot 的基本工作原理，此外还有一系列其他功能。

以下是部署 Ocelot 时使用的配置。

**基本实现**

![图片](./md_img/OcelotBasic.jpg '图片')

**结合IdentityServer**

![图片](./md_img/OcelotIndentityServer.jpg '图片')

**多实例**

![图片](./md_img/OcelotMultipleInstances.jpg '图片')

**结合Consul**

![图片](./md_img/OcelotMultipleInstancesConsul.jpg '图片')

**结合Service Fabric**

![图片](./md_img/OcelotServiceFabric.jpg '图片')

**入门指南**

Ocelot 旨在与 ASP.NET 一起使用，目前支持 net6.0、net7.0 和 net8.0 框架。

### .NET 8.0

#### 安装 NuGet 包
使用 NuGet 安装 Ocelot 及其依赖项。你需要创建一个 ASP.NET Core 最小 API 项目，并引入该包。然后按照下面的启动步骤和配置部分进行操作以完成设置。

```bash
Install-Package Ocelot
```

所有版本都可以在 NuGet Gallery 中找到 | Ocelot。

### 配置
以下是一个非常基础的 `ocelot.json`。它不会执行任何操作，但可以启动 Ocelot。

```json
{
    "Routes": [],
    "GlobalConfiguration": {
        "BaseUrl": "https://api.mybusiness.com"
    }
}
```

如果你想要一个能实际运行的例子，请使用以下配置：

```json
{
    "Routes": [
        {
        "DownstreamPathTemplate": "/todos/{id}",
        "DownstreamScheme": "https",
        "DownstreamHostAndPorts": [
            {
                "Host": "jsonplaceholder.typicode.com",
                "Port": 443
            }
        ],
        "UpstreamPathTemplate": "/todos/{id}",
        "UpstreamHttpMethod": [ "Get" ]
        }
    ],
    "GlobalConfiguration": {
        "BaseUrl": "https://localhost:5000"
    }
}
```

这里最重要的是 `BaseUrl` 属性。Ocelot 需要知道它运行的 URL 以便进行 Header 的查找与替换，以及某些管理配置。当设置这个 URL 时，它应该是客户端将看到 Ocelot 运行的外部 URL。例如，如果你运行的是容器，Ocelot 可能运行在 `http://123.12.1.1:6543` 上，但前面有一个 nginx 在 `https://api.mybusiness.com` 上响应。在这种情况下，Ocelot 的 `BaseUrl` 应该是 `https://api.mybusiness.com`。

如果你使用的是容器，并且需要 Ocelot 在 `http://123.12.1.1:6543` 上响应客户端请求，你也可以这样做。不过，如果你部署了多个 Ocelot 实例，可能需要通过命令行脚本传递这个 URL。希望你使用的调度程序能够传递 IP 地址。

### Program.cs 配置

在你的 `Program.cs` 文件中，你需要包含以下代码：

- `AddOcelot()` 添加 Ocelot 所需的默认服务。
- `UseOcelot().Wait()` 设置所有 Ocelot 中间件。

```csharp
using System.IO;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.Hosting;
using Ocelot.DependencyInjection;
using Ocelot.Middleware;

namespace OcelotBasic
{
    public class Program
    {
        public static void Main(string[] args)
        {
            new WebHostBuilder()
            .UseKestrel()
            .UseContentRoot(Directory.GetCurrentDirectory())
            .ConfigureAppConfiguration((hostingContext, config) =>
            {
                config
                    .SetBasePath(hostingContext.HostingEnvironment.ContentRootPath)
                    .AddJsonFile("appsettings.json", true, true)
                    .AddJsonFile($"appsettings.{hostingContext.HostingEnvironment.EnvironmentName}.json", true, true)
                    .AddJsonFile("ocelot.json")
                    .AddEnvironmentVariables();
            })
            .ConfigureServices(s => {
                s.AddOcelot();
            })
            .ConfigureLogging((hostingContext, logging) =>
            {
                // 添加日志记录
            })
            .UseIISIntegration()
            .Configure(app =>
            {
                app.UseOcelot().Wait();
            })
            .Build()
            .Run();
        }
    }
}
```

`AddOcelot` 方法将默认的 ASP.NET 服务添加到依赖注入容器。你也可以使用 `AddOcelotUsingBuilder` 方法来自定义构建器。更多说明请参阅依赖注入功能中的 “AddOcelotUsingBuilder method” 部分。

**贡献指南**

欢迎提交 Pull 请求、问题以及评论！

你可以在 Ocelot Discussions 讨论区中发布你的想法和问题。

我们遵循开发流程，该流程在发布流程（Release Process）中有详细描述。

**不支持的功能**

Ocelot 不支持以下功能：

### 分块编码（Chunked Encoding）
Ocelot 始终会获取请求体的大小并返回 `Content-Length` 头部。如果你的用例需要分块编码，很抱歉，Ocelot 无法满足这一需求。

### 转发 Host 头部
你发送给 Ocelot 的 `Host` 头部不会被转发到下游服务。显然，这会导致很多问题 😟。

### Swagger 支持
贡献者曾多次尝试通过 Ocelot 的 `ocelot.json` 文件生成 `swagger.json`，但这不符合 Ocelot 团队的设计愿景。如果你想在 Ocelot 中使用 Swagger，需要自己编写 `swagger.json`，并在 `Startup.cs` 或 `Program.cs` 中手动配置。以下是一个示例，展示如何注册一个中间件来加载自定义的 `swagger.json`，并在 `/swagger/v1/swagger.json` 路径上返回它。然后，它会从 `Swashbuckle.AspNetCore` 包中注册 `SwaggerUI` 中间件：

```csharp
app.Map("/swagger/v1/swagger.json", b =>
{
    b.Run(async x => {
        var json = File.ReadAllText("swagger.json");
        await x.Response.WriteAsync(json);
    });
});

app.UseSwaggerUI(c =>
{
    c.SwaggerEndpoint("/swagger/v1/swagger.json", "Ocelot");
});

app.UseOcelot().Wait();
```

### 不支持 Swagger 的原因
我们认为在 Ocelot 中使用 Swagger 并不合适，主要原因如下：
1. 我们已经在 `ocelot.json` 文件中手动定义了路由。如果你希望开发者能够查看可用路由，可以直接与他们分享 `ocelot.json` 文件（这可以通过提供代码仓库访问等方式轻松完成），或者使用 Ocelot 管理 API 来查询 Ocelot 的配置。
  
2. 很多人将 Ocelot 配置为代理所有流量，例如 `/products/{everything}`，将请求转发到产品服务。如果你解析这样的路径并生成 Swagger 路径，实际上并不能准确描述可用的内容。此外，Ocelot 无法理解下游服务返回的模型，并且同一个端点可能返回多个不同的模型。Ocelot 也不知道在 `POST`、`PUT` 等请求中会使用哪些模型，这样会变得非常混乱。

3. 最后，`Swashbuckle` 包在运行时无法重新加载修改后的 `swagger.json`。Ocelot 的配置可以在运行时发生变化，因此 Swagger 和 Ocelot 之间的信息可能不匹配，除非我们自己实现 Swagger 😋。

### 建议
如果开发者想要一个简单的方式来测试 Ocelot API，我们建议使用 **Postman**。甚至可以编写一个工具，将 `ocelot.json` 映射到 Postman 的 JSON 规范，不过我们并不打算这样做。

**注意事项**

许多错误和问题（“gotchas”）与 Web 服务器托管场景相关。请根据你的 Web 服务器查看以下部署和 Web 托管常见用户场景。

### IIS
[Microsoft Learn: 在 Windows 上使用 IIS 托管 ASP.NET Core](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/)

我们不建议将 Ocelot 应用程序部署到 IIS 环境，但如果你决定这样做，请注意以下注意事项：

1. 当使用 ASP.NET Core 2.2 及更高版本，并且想要使用进程内托管时，将 `UseIISIntegration()` 替换为 `UseIIS()`，否则会出现启动错误。

2. 确保使用进程外托管模型，而不是进程内托管模型（参见 [使用 IIS 和 ASP.NET Core 的进程外托管](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/iis/outsidetheprocess)），否则你可能会遇到非常慢的响应（参见问题 1657）。

3. 确保所有下游主机的 DNS 服务器在线并正常运行，否则你会遇到慢响应的问题（参见问题 1630）。

4. 社区经常报告与 IIS 相关的问题。如果你在 IIS 环境中托管 Ocelot 应用程序时遇到问题，首先阅读已打开/关闭的问题，然后在代码库中搜索 IIS。你可能会找到 Ocelot 社区成员提供的现成解决方案。

5. 最后，我们为所有与 IIS 相关的问题设置了特殊标签“`IIS`”。请随意在问题、PR、讨论等中使用这个标签。

### Kestrel
[Microsoft Learn: ASP.NET Core 中的 Kestrel Web 服务器](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel)

我们建议将 Ocelot 应用程序部署到自托管环境，例如 Kestrel 或 Docker。我们尝试优化 Ocelot Web 应用程序以适应 Kestrel 和 Docker 托管场景，但请注意以下注意事项：

1. **上传和下载大文件**：通过网关代理内容时，处理大（静态）文件时可能会出现问题。我们认为你的客户端应用程序应该直接集成到（静态）文件的持久存储和服务中，例如远程和分布式文件系统、CDN、静态文件和 Blob 存储等。由于性能原因，我们不建议通过网关传输大文件（100MB+ 或甚至更大的 1GB+），因为这会消耗内存和 CPU，造成长时间的延迟，产生下游流媒体网络错误，并影响其他路由。

   社区经常报告与大文件、`application/octet-stream` 内容类型、分块编码等相关的问题，请参见问题 749、1472。如果你仍然想通过 Ocelot 网关实例传输大文件，请使用 23.0 版本及更高版本 [1]。在出现错误的情况下，请查看下一个点。

2. **最大请求体大小**：如果应用程序实例未正确配置 Kestrel 的 `MaxRequestBodySize` 选项，并且上传了超出限制的不可预测大小的大文件，ASP.NET 的 `HttpRequest` 行为可能会异常。

   请查看这些文档：[最大请求体大小 | 配置 ASP.NET Core Kestrel Web 服务器选项](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel?view=aspnetcore-5.0#maximum-request-body-size)。

   作为快速修复，请使用以下配置示例：
   ```csharp
   builder.WebHost.ConfigureKestrel((context, serverOptions) =>
   {
       int myVideoFileMaxSize = 1_073_741_824; // 假设你的文件存储的最大文件大小为 1 GB (1_073_741_824)
       int totalSize = myVideoFileMaxSize + 26_258_176; // 并添加一些额外的大小
       serverOptions.Limits.MaxRequestBodySize = totalSize; // 1_100_000_000，因此 1 GB 文件不应超过限制
   });
   ```
   希望这能帮助你。

[1] 大文件传输在 23.0 版本中得到了稳定和完整的解决方案。我们相信我们的 PR 1724 和 1769 有助于解决 22.0.1 版本及更早版本中大内容代理问题。

**管理**

Ocelot 支持通过经过身份验证的 HTTP API 在运行时更改配置。身份验证可以通过两种方式进行：使用 Ocelot 内部的 IdentityServer（仅用于对管理 API 的请求进行身份验证），或者将管理 API 的身份验证集成到你自己的 IdentityServer 中。

### 使用你自己的 IdentityServer

如果你想集成到你自己的 IdentityServer，只需在 `ConfigureServices` 方法中添加以下配置选项和身份验证。之后，我们必须将这些选项传递给 `AddAdministration()` 扩展方法，该方法由 `AddOcelot()` 返回的 `OcelotBuilder` 使用，如下所示：

```csharp
public virtual void ConfigureServices(IServiceCollection services)
{
    Action<JwtBearerOptions> options = o =>
    {
        o.Authority = identityServerRootUrl;
        o.RequireHttpsMetadata = false;
        o.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateAudience = false,
        };
        // etc...
    };

    services
        .AddOcelot()
        .AddAdministration("/administration", options);
}
```

你现在需要从你的 IdentityServer 获取一个令牌，并在随后的请求中使用它来访问 Ocelot 的管理 API。

### 内部 IdentityServer

API 使用你从 Ocelot 自身请求的 Bearer 令牌进行身份验证。这是由 .NET 社区使用多年的 Identity Server 项目提供的。

为了启用管理部分，你需要做一些事情。首先，将以下内容添加到你的初始 `Startup.cs` 文件中：

路径可以是你想要的任何内容，建议不要使用你希望通过 Ocelot 路由的 URL，因为这将不起作用。管理部分使用 ASP.NET Core 的 `MapWhen` 功能，所有对 “{root}/administration” 的请求将被发送到这里，而不是 Ocelot 中间件。

`secret` 是 Ocelot 内部的 IdentityServer 用于对管理 API 请求进行身份验证的客户端密钥。这个密钥可以是你想要的任何值！为了将这个密钥字符串作为参数传递，我们必须调用 `AddAdministration()` 扩展方法，如下所示：

```csharp
public virtual void ConfigureServices(IServiceCollection services)
{
    services
        .AddOcelot()
        .AddAdministration("/administration", "secret");
}
```

为了使管理 API 正常工作，Ocelot / IdentityServer 必须能够调用自身进行验证。这意味着你需要将 Ocelot 的基本 URL 添加到全局配置中，如果它不是默认的 `http://localhost:5000`。请注意，如果你使用 Docker 托管 Ocelot，它可能无法回调到 localhost 等，你需要了解 Docker 网络的相关内容。无论如何，可以按以下方式完成：

如果你希望在本地的不同主机和端口上运行：

```json
"GlobalConfiguration": {
   "BaseUrl": "http://localhost:55580"
}
```

或者如果 Ocelot 通过 DNS 暴露：

```json
"GlobalConfiguration": {
   "BaseUrl": "http://mydns.com"
}
```

现在，如果你使用上述配置选项并希望访问 API，可以使用解决方案中的 `ocelot.postman_collection.json` Postman 脚本来更改 Ocelot 配置。显然，如果你在不同的 URL 上运行 Ocelot，你需要更改这些脚本。

这些脚本展示了如何从 Ocelot 请求 Bearer 令牌，然后使用它来 GET 当前配置和 POST 新配置。

### 集群中的多个 Ocelot 实例

如果你在集群中运行多个 Ocelot 实例，那么你需要使用证书来签名用于访问管理 API 的 Bearer 令牌。

为此，你需要为集群中的每个 Ocelot 添加两个环境变量：

- **OCELOT_CERTIFICATE**：用于签名令牌的证书路径。证书需要是 X509 类型，并且 Ocelot 需要能够访问它。
- **OCELOT_CERTIFICATE_PASSWORD**：证书的密码。

通常 Ocelot 使用临时签名凭证，但如果设置了这些环境变量，它将使用证书。如果集群中的所有其他 Ocelot 实例都使用相同的证书，那么一切就绪！

### 管理 API

- **POST {adminPath}/connect/token**
  这个请求用于获取用于管理区域的令牌，使用上面提到的客户端凭证。底层会调用 Ocelot 内部托管的 IdentityServer。

  请求体是表单数据格式，如下：

  ```plaintext
  client_id = admin
  client_secret = 设置时使用的值
  scope = admin
  grant_type = client_credentials
  ```

- **GET {adminPath}/configuration**
  这个请求获取当前的 Ocelot 配置。它与最初用于设置 Ocelot 的 JSON 完全相同。

- **POST {adminPath}/configuration**
  这个请求覆盖现有配置（可能应该是 PUT）。我们建议从 GET 端点获取配置，进行任何更改，然后将其 POST 回去… 很简单。

  请求体是 JSON 格式，与在文件系统上设置 Ocelot 时使用的 `FileConfiguration` 格式相同。

  请注意，如果你想使用此 API，运行 Ocelot 的进程必须有权限写入存储 `ocelot.json` 或 `ocelot.{environment}.json` 的磁盘。这是因为 Ocelot 在保存时会覆盖它们。

- **DELETE {adminPath}/outputcache/{region}**
  这个请求清除某个区域的缓存。如果你使用的是后端缓存，它会清除所有实例的缓存！这使得你能够在所有 Ocelot 实例中进行缓存并同时清除它们，因此建议使用分布式缓存。

  区域是你在 Ocelot 配置的 `FileCacheOptions` 部分中设置的 `Region` 字段值。

  `AddOcelot` 方法将默认的 ASP.NET 服务添加到依赖注入（DI）容器中。你可以在配置服务时调用另一个扩展的 `AddOcelotUsingBuilder` 方法，以开发你自己的自定义构建器。有关更多说明，请参见“AddOcelotUsingBuilder 方法”部分的依赖注入特性。

  ### 身份验证

要对路由进行身份验证并使用 Ocelot 的声明基础特性（如授权或根据令牌的值修改请求），用户必须在 `Startup.cs` 中注册身份验证服务，并为每个注册提供一个方案（身份验证提供程序密钥），例如：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    const string AuthenticationProviderKey = "MyKey";
    services
        .AddAuthentication()
        .AddJwtBearer(AuthenticationProviderKey, options =>
        {
            // 自定义身份验证设置
        });
}
```

在此示例中，`MyKey` 是注册该提供程序的方案。然后，我们将其映射到配置中的路由，使用以下 `AuthenticationOptions` 属性：

#### 单一密钥（即身份验证方案）

属性：`AuthenticationOptions.AuthenticationProviderKey`

我们将身份验证提供程序映射到配置中的路由，例如：

```json
"AuthenticationOptions": {
  "AuthenticationProviderKey": "MyKey",
  "AllowedScopes": []
}
```

当 Ocelot 运行时，它会查看此路由的 `AuthenticationProviderKey`，并检查是否有注册的身份验证提供程序。如果没有，Ocelot 将无法启动。如果有，路由将在执行时使用该提供程序。

如果路由需要身份验证，Ocelot 会在执行身份验证中间件时调用与其关联的方案。如果请求失败身份验证，Ocelot 将返回 HTTP 状态码 401 Unauthorized。

#### 多个身份验证方案

属性：`AuthenticationOptions.AuthenticationProviderKeys`

在实际的 ASP.NET 应用程序中，可能需要支持多种身份验证类型。要为每个适当的身份验证提供程序注册多个身份验证方案（身份验证提供程序密钥），可以使用并开发以下的抽象配置：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    const string DefaultScheme = JwtBearerDefaults.AuthenticationScheme; // Bearer
    services.AddAuthentication()
        .AddJwtBearer(DefaultScheme, options => { /* JWT 设置 */ })
        // 添加其他身份验证提供程序
        .AddMyProvider("MyKey", options => { /* 自定义身份验证设置 */ });
}
```

在此示例中，`MyKey` 和 `Bearer` 方案代表了这些提供程序注册的密钥。然后我们将这些方案映射到配置中的路由，如下所示：

```json
"AuthenticationOptions": {
  "AuthenticationProviderKeys": [ "Bearer", "MyKey" ], // 顺序很重要！
  "AllowedScopes": []
}
```

Ocelot 将应用所有为 `AuthenticationProviderKey` 定义的步骤，如单一密钥（即身份验证方案）。

注意，数组定义中的密钥顺序很重要！我们使用“第一个获胜”身份验证策略。

注册提供程序、初始化选项、转发身份验证工件可能是一个“真正”的编码挑战。如果你遇到困难或不知道该做什么，可以从我们的验收测试中寻找灵感（目前仅支持 Identity Server 4）。

### JWT 令牌

如果你想使用 JWT 令牌进行身份验证（例如来自 Auth0），你可以像正常一样注册身份验证中间件，例如：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    var authenticationProviderKey = "MyKey";
    services
        .AddAuthentication()
        .AddJwtBearer(authenticationProviderKey, options =>
        {
            options.Authority = "test";
            options.Audience = "test";
        });
    services.AddOcelot();
}
```

然后在配置中将身份验证提供程序密钥映射到路由，例如：

```json
"AuthenticationOptions": {
  "AuthenticationProviderKeys": [ "MyKey" ],
  "AllowedScopes": []
}
```

### Identity Server 承载令牌

要使用 IdentityServer 承载令牌，请按照常规方式在 `ConfigureServices` 中注册 IdentityServer 服务，并使用一个方案（密钥）。如果你不明白如何做，请参考 IdentityServer 文档。

```csharp
public void ConfigureServices(IServiceCollection services)
{
    var authenticationProviderKey = "MyKey";
    Action<JwtBearerOptions> options = (opt) =>
    {
        opt.Authority = "https://whereyouridentityserverlives.com";
        // ...
    };
    services
        .AddAuthentication()
        .AddJwtBearer(authenticationProviderKey, options);
    services.AddOcelot();
}
```

然后在配置中将身份验证提供程序密钥映射到路由，例如：

```json
"AuthenticationOptions": {
  "AuthenticationProviderKeys": [ "MyKey" ],
  "AllowedScopes": []
}
```

### Auth0 by Okta

另一种身份提供者是 Okta 的 Auth0，参见 [Auth0 开发者资源](https://auth0.com/docs)。

在启动配置中，添加以下内容：

```csharp
app.UseAuthentication()
    .UseOcelot().Wait();
```

在 `ConfigureServices` 方法中，至少添加以下内容：

```csharp
services
    .AddAuthentication()
    .AddJwtBearer(oktaProviderKey, options =>
    {
        options.Audience = configuration["Authentication:Okta:Audience"]; // Okta 授权服务器 Audience
        options.Authority = configuration["Authentication:Okta:Server"]; // Okta 授权发行者 URI URL，例如 https://{subdomain}.okta.com/oauth2/{authidentifier}
    });
services.AddOcelot(configuration);
```

注意：为了使 Ocelot 正确读取 Okta 的 `scope` 声明，你需要添加以下代码将默认的 Okta `"scp"` 声明映射到 `"scope"`：

```csharp
// 将 Okta "scp" 声明映射到 "scope"，以允许 Ocelot 读取/验证它们
JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Remove("scp");
JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Add("scp", "scope");
```

问题 446 包含了一些代码和示例，可能对 Okta 集成有帮助。

### 允许的作用域

如果你在 `AllowedScopes` 中添加作用域，Ocelot 将获取令牌中的所有用户声明，并确保用户具有列表中的至少一个作用域。这是一种按作用域限制对路由的访问的方法。

### 相关链接

- [Microsoft Learn: ASP.NET Core 身份验证和授权概述](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/)
- [Microsoft Learn: 在 ASP.NET Core 中使用特定方案授权](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/)
- [Microsoft Learn: ASP.NET Core 中的策略方案](https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies)
- [Microsoft .NET 博客: 使用 IdentityServer4 的 ASP.NET Core 身份验证](https://dev.to/dotnet/asp-net-core-authentication-with-identityserver4-43mc)

### 未来展望

我们鼓励您添加更多示例，如果您已经集成了其他身份提供者并且解决方案有效，请在仓库中开启“Show and Tell”讨论。

- **[1]** 使用 `AuthenticationProviderKeys` 属性代替 `AuthenticationProviderKey`。我们支持此过时属性以兼容旧版本和迁移目的。在未来的版本中，此属性可能会被移除，成为破坏性更改。
- **[2]** “多个身份验证方案”功能在问题 740 和 1580 中被请求，并作为 23.0 版本的一部分发布。
- **[3]** 我们希望您能提交新的 PR，为您的自定义场景添加额外的验收测试，特别是涉及多个身份验证方案的情况。

如有更多集成方案或使用经验，请在社区中分享，以帮助其他开发者更好地利用 Ocelot。

### 授权

Ocelot 支持基于声明的授权，这在认证后运行。这意味着，如果您有一个路由需要授权，您可以在路由配置中添加以下内容：

```json
"RouteClaimsRequirement": {
    "UserType": "registered"
}
```

在这个例子中，当调用 AuthorizationMiddleware 时，Ocelot 会检查用户是否具有声明类型 `UserType`，且该声明的值是否为 `"registered"`。如果不匹配，则用户不会被授权，响应将是 403 Forbidden。

#### 授权中间件

AuthorizationMiddleware 是 Ocelot 管道中的内置中间件。

- 之前的私有中间件: `ClaimsToClaimsMiddleware`
- 之前的公共中间件: `PreAuthorizationMiddleware`
- 当前的中间件: `AuthorizationMiddleware`
- 下一个私有中间件: `ClaimsToHeadersMiddleware`
- 下一个公共中间件: `PreQueryStringBuilderMiddleware`

中间件的调用顺序是：

`ClaimsToClaimsMiddleware → PreAuthorizationMiddleware → AuthorizationMiddleware → ClaimsToHeadersMiddleware → PreQueryStringBuilderMiddleware`

如您所知，从中间件注入部分，Authorization 中间件可以通过以下方式被覆盖：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    var configuration = new OcelotPipelineConfiguration
    {
        AuthorizationMiddleware = async (context, next) =>
        {
            await next.Invoke();
        }
    };
    app.UseOcelot(configuration);
}
```

这种覆盖方式应非常谨慎，因为覆盖 Authorization 中间件意味着您将失去通过路由的 `RouteClaimsRequirement` 属性进行的声明和范围授权。另一种选择是在实际授权之前，在 `PreAuthorizationMiddleware` 中进行准备工作，这个中间件是公共的并且可以被覆盖。

### 缓存

Ocelot 提供了一些基本的缓存功能，通过 CacheManager 项目实现。CacheManager 是一个解决许多缓存问题的优秀项目。我们建议使用这个包来与 Ocelot 一起进行缓存。

#### 安装

首先，添加以下 NuGet 包：

```bash
Install-Package Ocelot.Cache.CacheManager
```

这将提供 Ocelot 缓存管理器扩展方法的访问权限。

接下来，在 `ConfigureServices` 方法中进行如下配置：

```csharp
using Ocelot.Cache.CacheManager;

public void ConfigureServices(IServiceCollection services)
{
    services.AddOcelot()
        .AddCacheManager(x => x.WithDictionaryHandle());
}
```

#### 配置

为了在路由中使用缓存，需要在路由配置中添加以下设置：

```json
"FileCacheOptions": {
  "TtlSeconds": 15,
  "Region": "europe-central",
  "Header": "OC-Caching-Control",
  "EnableContentHashing": false
}
```

- **`TtlSeconds`**: 缓存的过期时间，以秒为单位。例如，设置为 15 秒表示缓存将在 15 秒后过期。
- **`Region`**: 缓存的区域标识。
- **`Header`**: 如果定义了头部名称（`Header` 属性），那么该头部的值会被用于缓存键的计算。如果头部值发生变化，缓存会失效。
- **`EnableContentHashing`**: 从版本 23.0 开始引入的属性。此选项控制是否将请求体的哈希值包括在缓存键中。由于请求体哈希可能会导致性能问题，因此默认情况下禁用。如果您的路由方法（如 POST、PUT）需要考虑请求体，请启用此选项。

#### 全局配置

从版本 23.3 开始，您可以在 `GlobalConfiguration` 中全局设置缓存属性，无需为每个路由单独配置。包括 Header 和 Region。但 TtlSeconds 仍需在每个路由中单独设置。

```json
"CacheOptions": {
  "EnableContentHashing": true
}
```

#### 自定义缓存

如果您想实现自定义的缓存方法，请实现以下接口并在 DI 容器中注册：

```csharp
services.AddSingleton<IOcelotCache<CachedResponse>, MyCache>();
```

- **`IOcelotCache<CachedResponse>`**: 用于输出缓存。
- **`IOcelotCache<FileConfiguration>`**: 用于缓存文件配置，特别是当从远程获取配置（如 Consul）时。

我们鼓励实现 Redis、Memcached 等缓存方法。如果您有实现方案，请在 Ocelot 的讨论区开设新的 Show and Tell 线程。

### 声明转换

Ocelot 允许用户访问声明，并将其转换为头部、查询字符串参数、其他声明，并更改下游路径。这些功能仅在用户经过身份验证后才可用。

在用户通过身份验证后，我们会运行声明到声明转换中间件（见 `ClaimsToClaimsMiddleware` 类）。这允许用户在调用授权中间件之前转换声明。用户授权后，我们会调用声明到头部中间件（见 `ClaimsToHeadersMiddleware` 类）、声明到查询字符串参数中间件（见 `ClaimsToQueryStringMiddleware` 类），最后是声明到下游路径中间件（见 `ClaimsToDownstreamPathMiddleware` 类）。

进行转换的语法对于每个过程都是相同的。在路由配置中，会添加一个带有特定名称的 JSON 字典，例如 `AddClaimsToRequest`、`AddHeadersToRequest`、`AddQueriesToRequest` 或 `ChangeDownstreamPathTemplate`。

**注意**: 这种语法并不是很理想，因此欢迎提出改进建议……

在这个字典中，条目指定了 Ocelot 应该如何进行转换！字典的键将成为声明、头部或查询参数的键。对于 `ChangeDownstreamPathTemplate`，键也必须在 `DownstreamPathTemplate` 中指定，以便进行转换。

条目的值被解析为执行转换的逻辑。首先，指定一个字典访问器，例如 `Claims[CustomerId]`。这意味着我们要访问声明并获取 `CustomerId` 声明类型。接下来是一个“更大于”符号 `>`，用于分割字符串。接下来的条目是值或带有索引的值。如果指定了值，Ocelot 将直接使用该值进行转换。如果值有索引，Ocelot 将查找在另一个“更大于”符号 `>` 后提供的分隔符。Ocelot 将基于分隔符分割值，并将请求的索引处的内容添加到转换中。

#### 声明到声明的转换

以下是一个将声明转换为声明的配置示例：

```json
"AddClaimsToRequest": {
    "UserType": "Claims[sub] > value[0] > |",
    "UserId": "Claims[sub] > value[1] > |"
}
```

这个配置示例展示了 Ocelot 如何查看用户的 `sub` 声明，并将其转换为 `UserType` 和 `UserId` 声明。假设 `sub` 的值为 `usertypevalue|useridvalue`。

#### 声明到头部的转换

以下是一个将声明转换为头部的配置示例：

```json
"AddHeadersToRequest": {
    "CustomerId": "Claims[sub] > value[1] > |"
}
```

这个配置示例展示了 Ocelot 如何查看用户的 `sub` 声明，并将其转换为 `CustomerId` 头部。假设 `sub` 的值为 `usertypevalue|useridvalue`。

#### 声明到查询字符串参数的转换

以下是一个将声明转换为查询字符串参数的配置示例：

```json
"AddQueriesToRequest": {
    "LocationId": "Claims[LocationId] > value"
}
```

这个配置示例展示了 Ocelot 如何查看用户的 `LocationId` 声明，并将其作为查询字符串参数转发到下游服务。

#### 声明到下游路径的转换

以下是一个将声明转换为下游路径自定义占位符的配置示例：

```json
"UpstreamPathTemplate": "/api/users/me/{everything}",
"DownstreamPathTemplate": "/api/users/{userId}/{everything}",
"ChangeDownstreamPathTemplate": {
    "userId": "Claims[sub] > value[1] > |"
}
```

这个配置示例展示了 Ocelot 如何查看用户的 `userId` 声明，并将其替换为 `DownstreamPathTemplate` 中的 `{userId}` 占位符。请注意，`ChangeDownstreamPathTemplate` 中指定的键必须与 `DownstreamPathTemplate` 中的占位符匹配。

**注意**: 如果 `ChangeDownstreamPathTemplate` 中指定的键在 `DownstreamPathTemplate` 中没有作为占位符存在，运行时将失败，并返回错误响应。

### 配置

可以在 `ocelot.json` 中找到一个示例配置。配置分为两个部分：`Routes` 数组和 `GlobalConfiguration`。

- **Routes** 是告诉 Ocelot 如何处理上游请求的对象。
- **GlobalConfiguration** 有点儿 hacky，它允许覆盖特定 Route 的设置。如果你不想管理大量的 Route 特定设置，这个配置很有用。

示例配置：

```json
{
  "Routes": [],
  "GlobalConfiguration": {}
}
```

以下是 Route 配置的示例。你不需要设置所有这些选项，但这是目前可用的所有选项：

```json
{
  "UpstreamPathTemplate": "/",
  "UpstreamHeaderTemplates": {}, // 字典
  "UpstreamHost": "",
  "UpstreamHttpMethod": [ "Get" ],
  "DownstreamPathTemplate": "/",
  "DownstreamHttpMethod": "",
  "DownstreamHttpVersion": "",
  "DownstreamHttpVersionPolicy": "",
  "AddHeadersToRequest": {},
  "AddClaimsToRequest": {},
  "RouteClaimsRequirement": {},
  "AddQueriesToRequest": {},
  "RequestIdKey": "",
  "FileCacheOptions": {
    "TtlSeconds": 0,
    "Region": "europe-central"
  },
  "RouteIsCaseSensitive": false,
  "ServiceName": "",
  "DownstreamScheme": "http",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 12345 }
  ],
  "QoSOptions": {
    "ExceptionsAllowedBeforeBreaking": 0,
    "DurationOfBreak": 0,
    "TimeoutValue": 0
  },
  "LoadBalancer": "",
  "RateLimitOptions": {
    "ClientWhitelist": [],
    "EnableRateLimiting": false,
    "Period": "",
    "PeriodTimespan": 0,
    "Limit": 0
  },
  "AuthenticationOptions": {
    "AuthenticationProviderKey": "",
    "AllowedScopes": []
  },
  "HttpHandlerOptions": {
    "AllowAutoRedirect": true,
    "UseCookieContainer": true,
    "UseTracing": true,
    "MaxConnectionsPerServer": 100
  },
  "DangerousAcceptAnyServerCertificateValidator": false,
  "SecurityOptions": {
    "IPAllowedList": [],
    "IPBlockedList": [],
    "ExcludeAllowedFromBlocked": false
  },
  "Metadata": {}
}
```

实际的 Route 属性模式可以在 C# 的 `FileRoute` 类中找到。如果你想了解如何使用这些选项，请继续阅读！

### 多个环境

像其他 ASP.NET Core 项目一样，Ocelot 支持类似 `appsettings.dev.json`、`appsettings.test.json` 等的配置文件名称。要实现这一点，请将以下内容添加到你的配置中：

```csharp
ConfigureAppConfiguration((context, config) =>
{
    var env = context.HostingEnvironment;
    config.SetBasePath(env.ContentRootPath)
        .AddJsonFile("appsettings.json", true, true)
        .AddJsonFile($"appsettings.{env.EnvironmentName}.json", true, true)
        .AddJsonFile("ocelot.json") // 主要配置文件
        .AddJsonFile($"ocelot.{env.EnvironmentName}.json") // 环境文件
        .AddEnvironmentVariables();
})
```

Ocelot 现在将使用环境特定的配置，如果没有找到相应的环境配置文件，则回退到 `ocelot.json`。

你还需要设置相应的环境变量，即 `ASPNETCORE_ENVIRONMENT`。更多信息可以在 ASP.NET Core 文档中找到：[在 ASP.NET Core 中使用多个环境](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/environments?view=aspnetcore-6.0)。

### 合并配置文件

这个功能允许用户拥有多个配置文件，从而更轻松地管理大型配置。

与直接添加配置（例如，使用 `AddJsonFile("ocelot.json")`）不同，你可以通过调用 `AddOcelot()` 来实现相同的结果，如下所示：

```csharp
ConfigureAppConfiguration((context, config) =>
{
    var env = context.HostingEnvironment;
    config.SetBasePath(env.ContentRootPath)
        .AddJsonFile("appsettings.json", true, true)
        .AddJsonFile($"appsettings.{env.EnvironmentName}.json", true, true)
        .AddOcelot(env) // 使用默认路径
        .AddEnvironmentVariables();
})
```

在这种情况下，Ocelot 将查找与模式 `^ocelot\.(.*?)\.json$` 匹配的文件，然后将它们合并在一起。如果你想设置 `GlobalConfiguration` 属性，你必须有一个名为 `ocelot.global.json` 的文件。

Ocelot 合并文件的方式基本上是加载它们，循环遍历它们，添加任何 Routes，添加任何 AggregateRoutes，如果文件名为 `ocelot.global.json`，则添加 GlobalConfiguration 以及任何 Routes 或 AggregateRoutes。Ocelot 然后将合并的配置保存到名为 `ocelot.json` 的文件中，并在 Ocelot 运行时将其作为真相源。

**注意1：** 目前，验证仅发生在 Ocelot 的最终配置合并期间。在排除问题时需要注意这一点。如果遇到问题，我们建议彻底检查 `ocelot.json` 文件的内容。

**注意2：** 合并功能仅在应用程序启动期间运行。因此，合并后的 `ocelot.json` 配置在合并和启动后保持静态。需要注意的是，`ConfigureAppConfiguration` 方法仅在 ASP.NET Web 应用程序的启动期间调用。一旦 Ocelot 应用程序启动，你不能调用 `AddOcelot` 方法，也不能在 `AddOcelot` 中使用合并功能。如果你仍需要实时更新主配置文件 `ocelot.json`，请参见 “响应配置更改” 部分。另外，请注意，使用 Administration API 进行的实时合并（例如 `ocelot.*.json`）尚未实现。

### 保持文件在文件夹中

你还可以给 Ocelot 一个特定的路径来查找配置文件，如下所示：

```csharp
ConfigureAppConfiguration((context, config) =>
{
    var env = context.HostingEnvironment;
    config.SetBasePath(env.ContentRootPath)
        .AddJsonFile("appsettings.json", true, true)
        .AddJsonFile($"appsettings.{env.EnvironmentName}.json", true, true)
        .AddOcelot("/my/folder", env) // 使用指定的文件夹路径
        .AddEnvironmentVariables();
})
```

Ocelot 需要 `HostingEnvironment` 以便它知道从合并算法中排除任何环境特定的内容。

### 将文件合并到内存中

默认情况下，Ocelot 将合并的配置写入磁盘，作为 `ocelot.json`（主要配置文件），将文件添加到 ASP.NET 配置提供程序中。

如果你的 Web 服务器没有对配置文件夹的写入权限，你可以指示 Ocelot 直接从内存中使用合并的配置。如下所示：

```csharp
config.AddOcelot(context.HostingEnvironment, MergeOcelotJson.ToMemory);
```

此功能在像 Azure、AWS 和 GCP 这样的云环境中尤为有用，特别是当应用程序缺乏足够的写入权限以保存文件时。此外，在 Docker 容器环境中，权限可能很少，可能需要大量的 DevOps 工作来启用文件写操作。因此，利用此功能可以节省时间！

### 配置更改的响应

如果你希望通过 Administration API 或重新加载 `ocelot.json` 来响应 Ocelot 配置的更改，请从 DI 容器中解析 `IOcelotConfigurationChangeTokenSource` 接口。你可以轮询更改令牌的 `IChangeToken.HasChanged` 属性，或者使用 `RegisterChangeCallback` 方法注册回调。

**轮询 HasChanged 属性**

```csharp
public class ConfigurationNotifyingService : BackgroundService
{
    private readonly IOcelotConfigurationChangeTokenSource _tokenSource;
    private readonly ILogger _logger;

    public ConfigurationNotifyingService(IOcelotConfigurationChangeTokenSource tokenSource, ILogger logger)
    {
        _tokenSource = tokenSource;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            if (_tokenSource.ChangeToken.HasChanged)
            {
                _logger.LogInformation("Configuration updated");
            }
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

**注册回调**

```csharp
public class MyDependencyInjectedClass : IDisposable
{
    private readonly IOcelotConfigurationChangeTokenSource _tokenSource;
    private readonly IDisposable _callbackHolder;

    public MyClass(IOcelotConfigurationChangeTokenSource tokenSource)
    {
        _tokenSource = tokenSource;
        _callbackHolder = tokenSource.ChangeToken.RegisterChangeCallback(_ => Console.WriteLine("Configuration changed"), null);
    }
    public void Dispose()
    {
        _callbackHolder.Dispose();
    }
}
```

### 下游Http版本

Ocelot 允许你选择它将用于代理请求的 HTTP 版本。它可以设置为 1.0、1.1 或 2

下游HTTP版本（DownstreamHttpVersion）  
Ocelot允许您选择用于发起代理请求的HTTP版本。它可以设置为1.0、1.1或2.0。

HttpVersion类

DownstreamHttpVersionPolicy [3]  
此路由属性使您能够在下游HTTP请求的HttpRequestMessage对象中配置VersionPolicy属性。有关更多详细信息，请参阅以下文档：

HttpRequestMessage.VersionPolicy属性  
HttpVersionPolicy枚举  
HttpVersion类

DownstreamHttpVersionPolicy选项与DownstreamHttpVersion设置密切相关。因此，仅仅指定DownstreamHttpVersion有时可能不够，尤其是当您的下游服务或Ocelot日志报告HTTP连接错误（如PROTOCOL_ERROR）时。在这些路由中，选择正确的DownstreamHttpVersionPolicy值对于HttpVersion策略至关重要，以防止此类协议错误。

### HTTP/2版本策略  
如果您希望确保Ocelot应用程序和启用SSL的下游服务之间的HTTP/2连接设置顺利进行：

```json
{
  "DownstreamScheme": "https",
  "DownstreamHttpVersion": "2.0",
  "DownstreamHttpVersionPolicy": "", // 空
  "DangerousAcceptAnyServerCertificateValidator": true
}
```

并且您使用以下代码片段配置全局设置以使用Kestrel：

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.WebHost.ConfigureKestrel(serverOptions =>
{
    serverOptions.ConfigureEndpointDefaults(listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http2;
    });
});
```

当所有组件都被设置为仅通过HTTP/2（非TLS，纯HTTP）进行通信时，下游服务可能会显示如下错误消息：

```
HTTP/2连接错误（PROTOCOL_ERROR）：无效的HTTP/2连接前言
```

为解决此问题，请确保HttpRequestMessage的VersionPolicy设置为RequestVersionOrHigher。因此，DownstreamHttpVersionPolicy应定义如下：

```json
{
  "DownstreamHttpVersion": "2.0",
  "DownstreamHttpVersionPolicy": "RequestVersionOrHigher" // ！
}
```

### 依赖注入  
Ocelot中该配置功能的依赖注入旨在在构建ASP.NET MVC管道服务之前，扩展和/或控制Ocelot内核的配置。主要方法是ConfigurationBuilderExtensions类中的AddOcelot方法，该类提供了多个具有对应签名的重载版本。

您可以在ASP.NET MVC网关应用程序（最简Web应用程序）的ConfigureAppConfiguration方法（位于Program.cs和Startup.cs中）中使用这些方法，来配置Ocelot管道和服务。

```csharp
namespace Microsoft.AspNetCore.Hosting;

public interface IWebHostBuilder
{
    IWebHostBuilder ConfigureAppConfiguration(Action<WebHostBuilderContext, IConfigurationBuilder> configureDelegate);
}
```

更多详情请参阅专门的配置概述部分以及依赖注入章节中的后续部分。

### 路由元数据  
Ocelot提供了多种功能，如路由、身份验证、缓存、负载均衡等。然而，有些用户可能会遇到Ocelot无法满足其特定需求的情况，或者他们希望自定义其行为。在这种情况下，Ocelot允许用户向路由配置中添加元数据。此属性可以存储用户可在中间件或委托处理程序中访问的任何任意数据。通过使用元数据，用户可以实现自己的逻辑并扩展Ocelot的功能。

以下是一个示例：

```json
{
  "Routes": [
      {
          "UpstreamHttpMethod": [ "GET" ],
          "UpstreamPathTemplate": "/posts/{postId}",
          "DownstreamPathTemplate": "/api/posts/{postId}",
          "DownstreamHostAndPorts": [
              { "Host": "localhost", "Port": 80 }
          ],
          "Metadata": {
              "api-id": "FindPost",
              "my-extension/param1": "overwritten-value",
              "other-extension/param1": "value1",
              "other-extension/param2": "value2",
              "tags": "tag1, tag2, area1, area2, func1",
              "json": "[1, 2, 3, 4, 5]"
          }
      }
  ],
  "GlobalConfiguration": {
      "Metadata": {
          "instance_name": "dc-1-54abcz",
          "my-extension/param1": "default-value"
      }
  }
}
```

现在，路由元数据可以通过DownstreamRoute对象访问：

```csharp
public static class OcelotMiddlewares
{
    public static Task PreAuthenticationMiddleware(HttpContext context, Func<Task> next)
    {
        var downstreamRoute = context.Items.DownstreamRoute();

        if(downstreamRoute?.Metadata is {} metadata)
        {
            var param1 = metadata.GetValueOrDefault("my-extension/param1") ?? throw new MyExtensionException("Param 1 is null");
            var param2 = metadata.GetValueOrDefault("my-extension/param2", "custom-value");

            // 处理元数据
        }

        return next();
    }
}
```

[1]  
“合并配置文件”功能是在问题296中请求的，从那时起我们在问题1216（PR 1227）中将其扩展为“合并文件到内存2”子功能，并作为版本23.2的一部分发布。

[2](1,2,3)  
“合并文件到内存2”子功能基于MergeOcelotJson枚举类型，具有ToFile和ToMemory的值。第一个默认是隐式的，而第二个正是您在合并到内存时所需要的。有关实现的更多详细信息，请参见ConfigurationBuilderExtensions类。

[3]  
“DownstreamHttpVersionPolicy 3”功能是在版本23.3中的问题1672中请求的。

## 委托处理程序（Delegating Handlers）  
Ocelot允许用户将委托处理程序添加到HttpClient传输层。这个功能是在问题208中请求的，团队认为它在多种情况下会非常有用。从那时起，我们在问题264中对其进行了扩展。

### 如何使用  
为了将委托处理程序添加到HttpClient传输层，需要完成两件主要的事情。

首先，要创建一个可以作为委托处理程序使用的类，其形式如下。我们将这些处理程序注册到ASP.NET Core的依赖注入（DI）容器中，因此您可以在处理程序的构造函数中注入已注册的其他服务。

```csharp
public class FakeHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request, CancellationToken token)
    {
        // 执行操作并可选地调用基处理程序...
        return await base.SendAsync(request, token);
    }
}
```

其次，您必须像下面这样将处理程序添加到DI容器中：

```csharp
ConfigureServices(s => s
    .AddOcelot()
    .AddDelegatingHandler<FakeHandler>()
    .AddDelegatingHandler<FakeHandlerTwo>()
)
```

这两个`AddDelegatingHandler`方法都有一个名为`global`的默认参数，默认为`false`。如果为`false`，则委托处理程序的意图是应用于特定的路由，通过`ocelot.json`配置（后面会详细介绍）。如果设置为`true`，它将成为全局处理程序，并应用于所有路由，如下所示：

```csharp
services.AddOcelot()
    .AddDelegatingHandler<FakeHandler>(true)
```

最后，如果您希望为特定路由设置委托处理程序，或者对特定和（或）全局委托处理程序进行排序（稍后会详细介绍），则必须在`ocelot.json`的特定路由中添加以下内容。数组中的名称必须与您的委托处理程序的类名匹配，以便Ocelot能够将它们正确匹配：

```json
"DelegatingHandlers": [
  "FakeHandlerTwo",
  "FakeHandler"
]
```

### 执行顺序  
您可以拥有任意数量的委托处理程序，它们将按以下顺序执行：

1. 所有未在`ocelot.json`的`DelegatingHandlers`数组中指定的全局处理程序，以它们在服务中添加的顺序执行。
2. 所有非全局的委托处理程序，以及任何在`DelegatingHandlers`数组中指定的全局处理程序，按它们在数组中的顺序执行。
3. 如果启用了追踪，则执行追踪委托处理程序（参见追踪文档）。
4. 如果启用了服务质量（QoS），则执行服务质量委托处理程序（参见服务质量文档）。
5. 最后，HttpClient发送HttpRequestMessage。

希望其他人也能觉得这个功能有用！

## 依赖注入  
命名空间: Ocelot.DependencyInjection  
源代码: DependencyInjection  

概述  
Ocelot 的依赖注入功能旨在扩展和/或控制将 Ocelot 核心作为 ASP.NET MVC 管道服务进行构建。  
ServiceCollectionExtensions 类的主要方法包括：  
- `AddOcelot`：将 Ocelot 所需的服务添加到 DI（依赖注入），并通过 `AddDefaultAspNetServices` 方法添加默认服务。  
- `AddOcelotUsingBuilder`：将 Ocelot 所需的服务添加到 DI，并通过隐式或显式注入配置，添加自定义的 ASP.NET 服务。  

在 ASP.NET MVC 网关应用程序（最小化 web 应用）的 `ConfigureServices` 方法（位于 `Program.cs` 和 `Startup.cs`）中使用 `IServiceCollection` 扩展方法来添加/构建 Ocelot 管道服务：

```csharp
namespace Microsoft.AspNetCore.Hosting;
public interface IWebHostBuilder
{
    IWebHostBuilder ConfigureServices(Action<IServiceCollection> configureServices);
}
```

实际上，`OcelotBuilder` 类是 Ocelot 的核心逻辑。  

### IServiceCollection 扩展
命名空间: Ocelot.DependencyInjection  
类: ServiceCollectionExtensions  
根据 `OcelotBuilder` 类的当前实现，`AddOcelot` 方法将所需的 ASP.NET 服务添加到 DI 容器中。您可以在配置服务时调用另一个更广泛的 `AddOcelotUsingBuilder` 方法，使用 `IMvcCoreBuilder` 对象来构建和使用自定义构建器。  

#### AddOcelot 方法  
签名：  
- `IOcelotBuilder AddOcelot(this IServiceCollection services);`  
- `IOcelotBuilder AddOcelot(this IServiceCollection services, IConfiguration configuration);`

这些 `IServiceCollection` 扩展方法通过隐式或显式注入配置，添加默认的 ASP.NET 服务和 Ocelot 应用程序服务。

注意！这两个方法通过 `AddDefaultAspNetServices` 方法在 Ocelot 管道中添加所需和默认的 ASP.NET 服务，这是默认构建器。在这种情况下，您只需调用 `AddOcelot` 方法即可，如果需要其他启动设置，通常会在功能章节中提到。该方法使您能够重用默认设置来构建 Ocelot 管道。替代方法是 `AddOcelotUsingBuilder` 方法，详见下一小节。  

#### AddOcelotUsingBuilder 方法  
签名：  
```csharp
using CustomBuilderFunc = System.Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder>;

IOcelotBuilder AddOcelotUsingBuilder(this IServiceCollection services, CustomBuilderFunc customBuilder);
IOcelotBuilder AddOcelotUsingBuilder(this IServiceCollection services, IConfiguration configuration, CustomBuilderFunc customBuilder);
```

这些 `IServiceCollection` 扩展方法通过隐式或显式注入配置，添加 Ocelot 应用程序服务并添加自定义 ASP.NET 服务。  

注意！该方法使用自定义构建器（即 `customBuilder` 参数）为 Ocelot 管道添加所需的自定义 ASP.NET 服务。建议阅读 `AddDefaultAspNetServices` 方法的文档，甚至查看实现，以了解默认的 ASP.NET 服务，它们是网关管道的最小组成部分。  

在这种自定义场景中，您可以在构建 ASP.NET MVC 管道期间控制所有内容，并提供自定义设置来构建 Ocelot 管道。

### OcelotBuilder 类  
源代码: Ocelot.DependencyInjection.OcelotBuilder  

`OcelotBuilder` 类是 Ocelot 的核心，执行以下操作：  
- 通过单个公共构造函数构建自身：
```csharp
public OcelotBuilder(IServiceCollection services, IConfiguration configurationRoot, Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder> customBuilder = null);
```
- 初始化并存储公共属性：`Services`（`IServiceCollection` 对象）、`Configuration`（`IConfiguration` 对象）和 `MvcCoreBuilder`（`IMvcCoreBuilder` 对象）。  
- 在构造阶段通过 `Services` 属性添加所有应用程序服务。  
- 通过构建器使用 `Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder>` 对象添加 ASP.NET 服务，分为以下两种开发场景：  
    - 默认构建器（`AddDefaultAspNetServices` 方法）如果未提供 `customBuilder` 参数。  
    - 自定义构建器，通过提供的委托对象作为 `customBuilder` 参数。  

### AddDefaultAspNetServices 方法  
类: OcelotBuilder  

该方法目前是受保护的，且禁止重写。其作用是通过 `IServiceCollection` 和 `IMvcCoreBuilder` 接口对象注入 Ocelot 管道最小部分所需的服务。

当前实现如下：
```csharp
protected IMvcCoreBuilder AddDefaultAspNetServices(IMvcCoreBuilder builder, Assembly assembly)
{
    Services
        .AddLogging()
        .AddMiddlewareAnalysis()
        .AddWebEncoders();

    return builder
        .AddApplicationPart(assembly)
        .AddControllersAsServices()
        .AddAuthorization()
        .AddNewtonsoftJson();
}
```

该方法无法被重写。它不是虚拟的，无法通过继承来覆盖当前行为。而且，当调用 `AddOcelot` 方法时，该方法是 Ocelot 管道的默认构建器。作为替代方案，要“覆盖”此默认构建器，你可以设计并重用自定义构建器，作为 `Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder>` 委托对象，并将其作为参数传递给 `AddOcelotUsingBuilder` 扩展方法。这样可以让你完全控制 Ocelot 管道的设计和构建，但在设计自定义 Ocelot 管道时需要小心，因为它是可定制的 ASP.NET MVC 管道。

**警告！** 管道的最小部分中大多数服务应当重用，只有少数服务可以移除。

**警告！！** 上述方法是在 ASP.NET MVC 管道构建时通过 `AddMvcCore` 方法在上层调用上下文中通过 `Services` 属性添加所需服务之后调用的。这些服务是 ASP.NET MVC 管道的绝对最低限度核心服务，必须始终添加到依赖注入 (DI) 容器中，它们在调用者上层上下文中调用该方法之前被隐式添加。因此，`AddMvcCore` 创建了一个 `IMvcCoreBuilder` 对象，并将其分配给 `MvcCoreBuilder` 属性。最后，作为默认构建器，上述方法接收 `IMvcCoreBuilder` 对象，并准备好进行进一步的扩展。

下一部分将向你展示通过自定义构建器设计 Ocelot 管道的示例。

### 自定义构建器

**目标：** 用 `System.Text.Json` 服务替换 `Newtonsoft.Json` 服务。

**问题：**
`AddOcelot` 方法通过默认构建器（`AddDefaultAspNetServices` 方法）中的 `AddNewtonsoftJson` 扩展方法添加了 Newtonsoft JSON 服务。`AddNewtonsoftJson` 方法是在旧的 .NET 和 Ocelot 版本中引入的，当时微软还没有推出 `System.Text.Json` 库，但现在它影响了正常使用，因此我们打算解决这个问题。

现代 JSON 服务开箱即用，能够通过 `JsonSerializerOptions` 属性配置 JSON 格式化程序在（反）序列化过程中的设置。

**解决方案：**
在 `ServiceCollectionExtensions` 类中，我们有以下方法：

```csharp
IOcelotBuilder AddOcelotUsingBuilder(this IServiceCollection services, Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder> customBuilder);
IOcelotBuilder AddOcelotUsingBuilder(this IServiceCollection services, IConfiguration configuration, Func<IMvcCoreBuilder, Assembly, IMvcCoreBuilder> customBuilder);
```

这些带有自定义构建器的方法允许你使用任何你想要的 JSON 库进行（反）序列化。但是我们打算创建支持 JSON 服务（例如 `System.Text.Json`）的自定义 `MvcCoreBuilder`。为此，我们需要在 `Startup.cs` 中调用 `MvcCoreMvcCoreBuilderExtensions` 类的 `AddJsonOptions` 扩展方法（NuGet 包：`Microsoft.AspNetCore.Mvc.Core`）：

```csharp
using Microsoft.Extensions.DependencyInjection;
using Ocelot.DependencyInjection;
using System.Reflection;

public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services
            .AddLogging()
            .AddMiddlewareAnalysis()
            .AddWebEncoders()
            // 添加自定义构建器
            .AddOcelotUsingBuilder(MyCustomBuilder);
    }

    private static IMvcCoreBuilder MyCustomBuilder(IMvcCoreBuilder builder, Assembly assembly)
    {
        return builder
            .AddApplicationPart(assembly)
            .AddControllersAsServices()
            .AddAuthorization()

            // 将 AddNewtonsoftJson() 替换为 AddJsonOptions()
            .AddJsonOptions(options =>
            {
                options.JsonSerializerOptions.WriteIndented = true; // 使用 System.Text.Json
            });
    }
}
```

示例代码提供了将 JSON 渲染为缩进文本的设置，而不是没有空格的压缩 JSON 文本。这只是一个常见的使用案例，你可以向构建器添加其他服务。

**配置概述**  
Ocelot的配置功能依赖注入旨在于构建ASP.NET MVC管道服务之前，扩展和/或控制Ocelot核心的配置。

要配置Ocelot管道和服务，可以在你的精简版Web应用的 `Program.cs` 和 `Startup.cs` 中使用 `IConfigurationBuilder` 扩展方法：

```csharp
namespace Microsoft.AspNetCore.Hosting;
public interface IWebHostBuilder
{
    IWebHostBuilder ConfigureAppConfiguration(Action<WebHostBuilderContext, IConfigurationBuilder> configureDelegate);
}
```

**IConfigurationBuilder扩展**  
命名空间：`Ocelot.DependencyInjection`  
类：`ConfigurationBuilderExtensions`

主要方法是 `ConfigurationBuilderExtensions` 类中的 `AddOcelot` 方法。这个方法有多个重载版本，带有相应的签名。

该方法的目的在于实际配置之前进行准备，包括以下步骤：

1. **合并部分JSON文件**：使用 `GetMergedOcelotJson` 方法合并部分JSON文件。
2. **选择合并类型**：你可以选择保存合并后JSON配置数据的方式，选项为 `ToFile` 或 `ToMemory`。
3. **框架扩展**：最后，该方法调用了以下本机 `IConfigurationBuilder` 框架扩展方法：

    - `AddJsonFile` 方法：在合并阶段之后，添加主要的配置文件（通常是 `ocelot.json`）。它使用 `ToFile` 合并类型选项将文件写回文件系统（默认情况下为此选项）。
    - `AddJsonStream` 方法：在合并阶段之后，将主要配置文件的JSON数据作为UTF-8流添加到内存中。它使用 `ToMemory` 合并类型选项。

### AddOcelot方法  
最常见版本的签名：

```csharp
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, IWebHostEnvironment env);
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, string folder, IWebHostEnvironment env);
```
注意：这些版本使用隐式的 `ToFile` 合并类型将 `ocelot.json` 写回磁盘，最终调用 `AddJsonFile` 扩展方法。

指定 `MergeOcelotJson` 选项的版本签名：

```csharp
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, IWebHostEnvironment env, MergeOcelotJson mergeTo,
    string primaryConfigFile = null, string globalConfigFile = null, string environmentConfigFile = null, bool? optional = null, bool? reloadOnChange = null);
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, string folder, IWebHostEnvironment env, MergeOcelotJson mergeTo,
    string primaryConfigFile = null, string globalConfigFile = null, string environmentConfigFile = null, bool? optional = null, bool? reloadOnChange = null);
```

注意：这些版本包括可选参数来指定合并操作涉及的三个主要文件的位置。理论上，这些文件可以位于任何地方，但实际上最好将它们放在同一个文件夹中。

指定自定义 `FileConfiguration` 对象的版本签名：

```csharp
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, FileConfiguration fileConfiguration,
    string primaryConfigFile = null, bool? optional = null, bool? reloadOnChange = null);
IConfigurationBuilder AddOcelot(this IConfigurationBuilder builder, FileConfiguration fileConfiguration, IWebHostEnvironment env, MergeOcelotJson mergeTo,
    string primaryConfigFile = null, string globalConfigFile = null, string environmentConfigFile = null, bool? optional = null, bool? reloadOnChange = null);
```

注意1：这些版本包括可选参数来指定合并操作涉及的三个主要文件的位置。  
注意2：你的 `FileConfiguration` 对象可以从任何地方进行序列化/反序列化：本地或远程存储、Consul KV存储，甚至是数据库。有关此功能的更多信息，请阅读PR 1569。

动态配置功能是在issues 1228和1235中提出的需求，已通过PR 1569在20.0版本中实现。自此之后，我们通过PR 1227进行了扩展，并在23.2版本中发布。

## 错误状态码  
Ocelot 会根据内部逻辑在某些情况下返回 HTTP 状态错误码：

**客户端错误响应**  
- **401** - 如果认证中间件运行时用户未认证。
- **403** - 如果授权中间件运行时用户未认证、声明值未授权、范围未授权、用户没有所需声明或找不到声明。
- **404** - 如果无法找到下游路由，或 Ocelot 无法将内部错误码映射到 HTTP 状态码。
- **499** - 如果请求被客户端取消。

**服务器错误响应**  
- **500** - 如果无法完成对下游服务的 HTTP 请求，且异常不是 `OperationCanceledException` 或 `HttpRequestException`。
- **502** - 如果无法连接到下游服务。
- **503** - 如果下游请求超时。

**设计**  
历史上，Ocelot 错误由 `HttpExceptionToErrorMapper` 类实现。`Map` 方法将 `System.Exception` 对象转换为 Ocelot 原生的 `Errors.Error` 对象。

我们对 HTTP 状态码进行了覆盖，因为异常到错误的映射。这可能会让开发人员感到困惑，因为下游服务的实际状态码可能不同并且会丢失。请研究和审查所有上游服务的响应头。如果没有找到状态码和（或）所需的头部信息，`Headers Transformation` 功能应能提供帮助。

我们希望你能在我们仓库的讨论区分享你的用例。

## GraphQL

Ocelot 并不直接支持 GraphQL，但由于许多人对此提出了需求，我们希望展示如何轻松集成 .NET 的 GraphQL 库。

请查看示例项目 [OcelotGraphQL](https://github.com/YourRepo/OcelotGraphQL)。通过结合使用 graphql-dotnet 项目和 Ocelot 的 Delegating Handlers 特性，这一集成过程相当简单。然而，目前我们不打算与 GraphQL 更紧密地集成。请查看示例的 README.md，这将为你提供足够的指导来实现这个集成！

**未来计划**

如果你对 GraphQL 和提到的 .NET 包有足够的经验，我们欢迎你对示例进行贡献。谁知道呢，也许你会受到示例开发的启发，并提出一些设计方案，作为在 Ocelot 中实现 GraphQL 特性的粗略草案。祝好运！

同时，欢迎来到仓库的讨论区！

## Headers Transformation

Ocelot 允许用户在下游请求之前和之后转换头部。目前，Ocelot 仅支持查找和替换功能。这个特性在 issue 190 中被请求，团队决定这个功能在各种情况下都很有用。

**添加到请求**

这个特性在 issue 313 中被请求。

如果你想在上游请求中添加一个头部，请在你的 `ocelot.json` 中的 Route 配置中添加以下内容：

```json
"UpstreamHeaderTransform": {
  "Uncle": "Bob"
}
```

在上面的例子中，一个键为 `Uncle`、值为 `Bob` 的头部将被发送到上游服务。

占位符也被支持（见下文）。

**添加到响应**

这个特性在 issue 280 中被请求。

如果你想在下游响应中添加一个头部，请在 `ocelot.json` 中的 Route 配置中添加以下内容：

```json
"DownstreamHeaderTransform": {
  "Uncle": "Bob"
}
```

在上面的例子中，一个键为 `Uncle`、值为 `Bob` 的头部将由 Ocelot 返回给请求的特定 Route。

如果你想返回 Butterfly APM 的 trace id，可以使用以下配置：

```json
"DownstreamHeaderTransform": {
  "AnyKey": "{TraceId}"
}
```

**查找和替换**

为了转换头部，首先我们指定头部的键，然后指定我们想要的转换类型。例如：

```json
"Test": "http://www.bbc.co.uk/, http://ocelot.com/"
```

键为 `Test`，值为 `http://www.bbc.co.uk/, http://ocelot.com/`。值表示：将 `http://www.bbc.co.uk/` 替换为 `http://ocelot.com/`。语法为 `{find}, {replace}`。这应该比较简单。下面的示例进一步解释了这个过程。

**下游请求前**

要在 `ocelot.json` 中的 Route 配置中替换 `http://www.bbc.co.uk/` 为 `http://ocelot.com/`，请添加以下内容。这一头部将在请求下游之前被更改，并发送到下游服务器。

```json
"UpstreamHeaderTransform": {
  "Test": "http://www.bbc.co.uk/, http://ocelot.com/"
}
```

**下游请求后**

要在 `ocelot.json` 中的 Route 配置中替换 `http://www.bbc.co.uk/` 为 `http://ocelot.com/`，请添加以下内容。这一转换将在 Ocelot 接收到下游服务的响应后进行。

```json
"DownstreamHeaderTransform": {
  "Test": "http://www.bbc.co.uk/, http://ocelot.com/"
}
```

**占位符**

Ocelot 允许在头部转换中使用占位符。

- `{BaseUrl}` - 使用 Ocelot 基本 URL，例如 `http://localhost:5000` 作为其值。
- `{DownstreamBaseUrl}` - 使用下游服务的基本 URL，例如 `http://localhost:5000` 作为其值。目前这仅适用于 `DownstreamHeaderTransform`。
- `{RemoteIpAddress}` - 查找客户端的 IP 地址，使用 `IHttpContextAccessor.HttpContext.Connection.RemoteIpAddress.ToString()`，因此你将获得一些 IP 地址。更多信息请查看 `GetRemoteIpAddress` 方法。
- `{TraceId}` - 使用 Butterfly APM 的 Trace Id。目前这仅适用于 `DownstreamHeaderTransform`。
- `{UpstreamHost}` - 查找传入的 Host 头部。

目前，我们认为这些占位符足以满足基本用户场景。如果你需要更多占位符，可以关注未来的更新。

**处理 302 重定向**

Ocelot 默认会自动跟随重定向，但如果你想将 `Location` 头部返回给客户端，可能需要将 `Location` 更改为 Ocelot 而不是下游服务。Ocelot 允许通过以下配置实现这一点：

```json
"DownstreamHeaderTransform": {
  "Location": "http://www.bbc.co.uk/, http://ocelot.com/"
},
"HttpHandlerOptions": {
  "AllowAutoRedirect": false
}
```

或者，你可以使用 `{BaseUrl}` 占位符：

```json
"DownstreamHeaderTransform": {
  "Location": "http://localhost:6773, {BaseUrl}"
},
"HttpHandlerOptions": {
  "AllowAutoRedirect": false
}
```

最后，如果你使用了负载均衡器与 Ocelot 结合使用，你会得到多个下游基本 URL，所以上述方法可能无法正常工作。在这种情况下，你可以这样做：

```json
"DownstreamHeaderTransform": {
  "Location": "{DownstreamBaseUrl}, {BaseUrl}"
},
"HttpHandlerOptions": {
  "AllowAutoRedirect": false
}
```

**X-Forwarded-For**

使用 `{RemoteIpAddress}` 占位符的示例：

```json
"UpstreamHeaderTransform": {
  "X-Forwarded-For": "{RemoteIpAddress}"
}
```

## Kubernetes [1] aka K8s

A part of feature: Service Discovery [2]

Ocelot 可以调用 K8s 的 endpoints API 来获取指定命名空间内的所有端点，并在这些端点之间进行负载均衡。Ocelot 曾经使用服务 API 来向 K8s 服务发送请求，但由于服务没有按预期进行负载均衡，这一做法在 PR 1134 中被更改。

**安装**

首先，你需要安装提供 Kubernetes 支持的 NuGet 包：

```bash
Install-Package Ocelot.Provider.Kubernetes
```

然后，将以下内容添加到你的 `ConfigureServices` 方法中：

```csharp
services.AddOcelot().AddKubernetes();
```

如果你在 Kubernetes 中部署了服务，通常会使用命名服务来访问它们。默认情况下，`usePodServiceAccount` 设置为 `true`，这意味着使用 Pod 的 Service Account 来访问 K8s 集群的服务，需要基于 RBAC 授权的 Service Account：

```csharp
public static class OcelotBuilderExtensions
{
    public static IOcelotBuilder AddKubernetes(this IOcelotBuilder builder, bool usePodServiceAccount = true);
}
```

你可以通过 RBAC 角色绑定（参见 Permissive RBAC Permissions）来复制一个 Permissive，K8s API 服务器和令牌将从 Pod 中读取。

```bash
kubectl create clusterrolebinding permissive-binding --clusterrole=cluster-admin --user=admin --user=kubelet --group=system:serviceaccounts
```

**配置**

以下示例展示了如何设置一个在 Kubernetes 中工作的 Route。最重要的是 `ServiceName`，它由 Kubernetes 服务名称组成。我们还需要在 `GlobalConfiguration` 中设置 `ServiceDiscoveryProvider`。

**Kube 默认提供程序**

以下是一个典型的配置示例：

```json
"Routes": [
  {
    "ServiceName": "downstreamservice",
    // ...
  }
],
"GlobalConfiguration": {
  "ServiceDiscoveryProvider": {
    "Host": "192.168.0.13",
    "Port": 443,
    "Token": "txpc696iUhbVoudg164r93CxDTrKRVWG",
    "Namespace": "Dev",
    "Type": "Kube"
  }
}
```

服务部署在命名空间 Dev 中，`ServiceDiscoveryProvider` 类型为 Kube，你也可以设置 `PollKube` 提供程序类型。

注意 1: Host, Port 和 Token 不再使用。

注意 2: Kube 提供程序通过 `ServiceName` 查找服务条目，然后从 `EndpointSubsetV1.Ports` 集合中检索第一个可用端口。因此，如果未指定端口名称，默认的下游协议将是 http。

**PollKube 提供程序**

如果你希望 Ocelot 定期轮询 Kubernetes 以获取最新的服务信息而不是每次请求时（默认行为），你需要设置以下配置：

```json
"ServiceDiscoveryProvider": {
  "Namespace": "dev",
  "Type": "PollKube",
  "PollingInterval": 100 // ms
}
```

轮询间隔以毫秒为单位，告诉 Ocelot 轮询 Kubernetes 获取服务配置更改的频率。

请注意，这里有权衡。如果你轮询 Kubernetes，Ocelot 可能无法知道服务是否已经宕机，具体取决于你的轮询间隔，你可能会遇到比每次请求时更多的错误。这主要取决于你的服务的波动性。对于大多数人来说，这可能不会成为问题，轮询可能会比每次请求时调用 Kubernetes 稍微提高性能。Ocelot 无法为你解决这些问题。

**全局与路由级别**

如果你的下游服务位于不同的命名空间，你可以通过在 Route 级别指定 `ServiceNamespace` 来覆盖全局设置：

```json
"Routes": [
  {
    "ServiceName": "downstreamservice",
    "ServiceNamespace": "downstream-namespace"
  }
]
```

**下游方案与端口名称 [3]**

Kubernetes 配置允许为每个端点子集的地址定义多个端口及其名称。在绑定多个端口时，你需要为每个子集端口分配一个名称。为了使 Kube 提供程序能够通过名称识别所需的端口，你需要指定 `DownstreamScheme` 以便提供程序通过端口名称进行比较；如果未指定，集合中的第一个端口条目将被选择为默认值。

例如，考虑一个在 Kubernetes 上暴露了两个端口的服务：443 的 https 和 80 的 http，如下所示：

```
Name:         my-service
Namespace:    default
Subsets:
  Addresses:  10.1.161.59
  Ports:
    Name   Port  Protocol
    ----   ----  --------
    https  443   TCP
    http   80    TCP
```

当你需要使用 http 端口而故意绕过默认的 https 端口（第一个端口）时，你必须定义 `DownstreamScheme` 以使提供程序能够识别所需的 http 端口，具体如下：

```json
"Routes": [
  {
    "ServiceName": "my-service",
    "DownstreamScheme": "http" // 端口名称 -> http -> 端口是 80
  }
]
```

注意：在没有指定 `DownstreamScheme` 的情况下（这是默认行为），Kube 提供程序将从 `EndpointSubsetV1.Ports` 集合中选择第一个可用的端口。因此，如果未指定端口名称，默认的下游协议将是 http。

[1](1,2)
Wikipedia | K8s 网站 | K8s 文档 | K8s GitHub

[2]
该特性是作为 issue 345 的一部分请求的，旨在添加对 Kubernetes 服务发现提供程序的支持。

[3]
“下游方案与端口名称”特性是作为 issue 1967 的一部分请求的，并在版本 23.3 中发布。

## 负载均衡器*

Ocelot 可以在每个路由中对可用的下游服务进行负载均衡。这意味着你可以扩展下游服务，Ocelot 可以有效地使用它们。可用的负载均衡器类型有：

- **LeastConnection**（最少连接）：跟踪哪些服务正在处理请求，并将新请求发送到连接最少的服务。该算法的状态不会在 Ocelot 集群中分布。

- **RoundRobin**（轮询）：循环遍历可用的服务并发送请求。该算法的状态不会在 Ocelot 集群中分布。

- **NoLoadBalancer**（无负载均衡器）：从配置或服务发现中选择第一个可用的服务。

- **CookieStickySessions**（Cookie 粘性会话）：使用 Cookie 将所有请求固定到特定的服务器。更多信息请见下文。

你必须在配置中选择要使用的负载均衡器。

**配置**

以下示例展示了如何使用 `ocelot.json` 配置多个下游服务，并选择 LeastConnection 负载均衡器。这是设置负载均衡的最简单方法。

```json
{
  "UpstreamPathTemplate": "/posts/{postId}",
  "UpstreamHttpMethod": [ "Put", "Delete" ],
  "DownstreamPathTemplate": "/api/posts/{postId}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "10.0.1.10", "Port": 5000 },
    { "Host": "10.0.1.11", "Port": 5000 }
  ],
  "LoadBalancerOptions": {
    "Type": "LeastConnection"
  }
}
```

**服务发现**

以下示例展示了如何使用服务发现配置路由，然后选择 LeastConnection 负载均衡器。

```json
{
  // ...
  "ServiceName": "product",
  "LoadBalancerOptions": {
    "Type": "LeastConnection"
  }
}
```

配置完成后，Ocelot 将从服务发现提供程序中查找下游主机和端口，并在所有可用服务之间进行负载均衡。如果你从服务发现提供程序（如 Consul）中添加或移除服务，Ocelot 应该会相应地停止调用已移除的服务，并开始调用已添加的服务。

**Cookie 粘性会话类型**

我们实现了一个非常基本的粘性会话负载均衡器类型。它的使用场景是你有一组下游服务器，它们不共享会话状态，因此如果有多个请求到达其中一台服务器，则应始终发送到相同的服务器，否则用户的会话状态可能会出错。这个功能是应 Issue 322 的要求实现的，尽管用户希望的功能比仅仅的粘性会话更复杂，但我们认为这个功能仍然很有用！

要设置 Cookie 粘性会话负载均衡器，你需要进行如下配置：

```json
{
  "UpstreamPathTemplate": "/posts/{postId}",
  "UpstreamHttpMethod": [ "Put", "Delete" ],
  "DownstreamPathTemplate": "/api/posts/{postId}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "10.0.1.10", "Port": 5000 },
    { "Host": "10.0.1.11", "Port": 5000 }
  ],
  "LoadBalancerOptions": {
    "Type": "CookieStickySessions",
    "Key": "ASP.NET_SessionId",
    "Expiry": 1800000
  }
}
```

**LoadBalancerOptions** 配置项包括：

- **Type**：必须设置为 `CookieStickySessions`。
- **Key**：要用于粘性会话的 Cookie 键。
- **Expiry**：会话保持的时间（以毫秒为单位）。每次请求时会刷新这个时间，这模仿了会话通常的工作方式。

如果你有多个路由使用相同的 LoadBalancerOptions，那么所有这些路由将使用相同的负载均衡器进行后续请求。这意味着会话将在路由之间粘性。

请注意，如果你指定了多个 `DownstreamHostAndPort` 或者使用了如 Consul 这样的服务发现提供程序，并且返回了多个服务，则 Cookie 粘性会话会使用轮询来选择下一个服务器。目前这是硬编码的，但可能会有所改变。

**自定义负载均衡器**

David Lievrouw 在 PR 1155（其 Issue 961）中实现了为 Ocelot 提供自定义负载均衡器的方法。

要创建和使用自定义负载均衡器，你可以按以下步骤操作。以下是一个基本的负载均衡配置，类型设置为 `CustomLoadBalancer`，这是我们将设置的一个类来实现负载均衡。

```json
{
  "UpstreamPathTemplate": "/posts/{postId}",
  "UpstreamHttpMethod": [ "Put", "Delete" ],
  "DownstreamPathTemplate": "/api/posts/{postId}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "10.0.1.10", "Port": 5000 },
    { "Host": "10.0.1.11", "Port": 5000 }
  ],
  "LoadBalancerOptions": {
    "Type": "CustomLoadBalancer"
  }
}
```

然后你需要创建一个实现了 `ILoadBalancer` 接口的类。以下是一个简单的轮询示例：

```csharp
public class CustomLoadBalancer : ILoadBalancer
{
    private readonly Func<Task<List<Service>>> _services;
    private readonly object _lock = new object();
    private int _last;

    public CustomLoadBalancer(Func<Task<List<Service>>> services)
    {
        _services = services;
    }

    public async Task<Response<ServiceHostAndPort>> Lease(HttpContext httpContext)
    {
        var services = await _services?.Invoke();
        lock (_lock)
        {
            if (_last >= services.Count)
                _last = 0;

            var next = services[_last++];
            return new OkResponse<ServiceHostAndPort>(next.HostAndPort);
        }
    }

    public void Release(ServiceHostAndPort hostAndPort) { }
}
```

最后，你需要将这个类注册到 Ocelot 中。

我们使用了下面的最复杂的示例来展示所有可以传递给工厂的数据/类型：

```csharp
Func<IServiceProvider, DownstreamRoute, IServiceDiscoveryProvider, CustomLoadBalancer> loadBalancerFactoryFunc =
    (serviceProvider, Route, serviceDiscoveryProvider) => new CustomLoadBalancer(serviceDiscoveryProvider.Get);

services.AddOcelot()
    .AddCustomLoadBalancer(loadBalancerFactoryFunc);
```

不过，也有一个更简单的示例效果相同：

```csharp
services.AddOcelot()
    .AddCustomLoadBalancer<CustomLoadBalancer>();
```

有多个扩展方法可以添加自定义负载均衡器，接口如下：

- `IOcelotBuilder AddCustomLoadBalancer<T>()` 其中 T 需要实现 `ILoadBalancer`，并且需要一个默认构造函数。
- `IOcelotBuilder AddCustomLoadBalancer<T>(Func<T> loadBalancerFactoryFunc)` 其中 T 需要实现 `ILoadBalancer`。
- `IOcelotBuilder AddCustomLoadBalancer<T>(Func<IServiceProvider, T> loadBalancerFactoryFunc)` 其中 T 需要实现 `ILoadBalancer`。
- `IOcelotBuilder AddCustomLoadBalancer<T>(Func<DownstreamRoute, IServiceDiscoveryProvider, T> loadBalancerFactoryFunc)` 其中 T 需要实现 `ILoadBalancer`。
- `IOcelotBuilder AddCustomLoadBalancer<T>(Func<IServiceProvider, DownstreamRoute, IServiceDiscoveryProvider, T> loadBalancerFactoryFunc)` 其中 T 需要实现 `ILoadBalancer`。

启用自定义负载均衡器时，Ocelot 会根据负载均衡器的类名查找你的负载均衡器。如果找到匹配的类，它将使用你的负载均衡器进行负载均衡。如果 Ocelot 无法匹配配置中的负载均衡器类型与已注册的负载均衡器类的名称，你将收到 HTTP 500 内部服务器错误。如果你的负载均衡器工厂在 Ocelot 调用时抛出异常，你也将收到 HTTP 500 内部服务器错误。

请记住，如果你在配置中没有指定负载均衡器，Ocelot 将不会尝试进行负载均衡。

## 日志记录

Ocelot 目前使用标准日志接口 `ILoggerFactory` 和 `ILogger<T>`。这封装在 `IOcelotLogger` 和 `IOcelotLoggerFactory` 中，当前的实现基于标准的 ASP.NET Core 日志功能。这样做的原因是 Ocelot 可以在日志中添加一些额外的信息，例如如果配置了 `RequestId`。

Ocelot 有一个全局错误处理中间件，它应该会捕获抛出的任何异常并将其记录为错误。

最后，如果日志级别设置为 Trace，Ocelot 会记录启动、完成以及任何抛出异常的中间件，这可能会非常有用。

**请求 ID**

不使用标准框架日志的原因是，我们无法找到如何覆盖设置 `IncludeScopes` 为 `true` 时记录的 `RequestId`。接下来是相关的功能。

每条日志记录都有以下两个属性：

- **RequestId**：当前请求的 ID，以纯字符串表示，例如 0HMVD33IIJRFR:00000001
- **PreviousRequestId**：前一个请求的 ID

作为注入到服务类构造函数中的 `IOcelotLogger` 接口对象，当前默认的 Ocelot 日志记录器 (`OcelotLogger` 类) 从 `IRequestScopedDataRepository` 接口对象中读取这两个属性。有关这些属性和请求 ID 日志记录功能的其他详细信息，请参见请求 ID 章节。

**警告**

如果你将日志记录到 MS Console，性能可能会非常差。团队在 Ocelot 性能问题上遇到了许多问题，并且通常使用 Debug 级别进行日志记录，记录到 Console 中。

警告！确保在生产环境中将日志记录到合适的地方！

- 在生产环境中使用 Error 和 Critical 级别！
- 在测试和预生产环境中使用 Warning 级别！

以下是最佳实践部分的其他建议。

**最佳实践**

- **首先**  
  确保在配置日志时设置了最低日志级别。最低日志级别在应用程序的 `appsettings.json` 文件中设置。该级别在 `Logging` 部分定义，例如：

  ```json
  {
    "Logging": {
      "LogLevel": {
        "Default": "Information",
        "Microsoft.AspNetCore": "Warning"
      }
    }
  }
  ```

  无论是使用 Serilog 还是标准的 Microsoft 提供程序，日志配置将从这一部分中获取。

  ```csharp
  .ConfigureAppConfiguration((_, config) =>
  {
      config.AddJsonFile($"appsettings.{env.EnvironmentName}.json", false, false);
      // ...
  })
  ```

  但是，有一点需要注意。可以使用 `SetMinimumLevel()` 方法定义最低日志级别。小心确保只在一个地方设置日志级别，例如：

  ```csharp
  ConfigureLogging(logging =>
  {
      logging.ClearProviders();
      logging.SetMinimumLevel(minLogLevel);
      logging.AddConsole(); // 仅用于开发和/或测试环境
  })
  ```

  还请使用 `ClearProviders()` 方法，以确保仅考虑你希望使用的提供程序，如上述示例中的 Console。

- **其次**  
  确保为每个环境（开发、测试、生产等）使用适当的最低日志级别。因此，请再次阅读警告部分的重要说明！

- **第三**  
  Ocelot 在 22.0 版本中改进了日志记录：现在可以使用工厂方法来生成消息字符串，只有在最低日志级别允许时才会执行。

  例如，假设有一条包含多个变量的消息，只有在最低日志级别为 Debug 时才生成。如果最低日志级别为 Warning，则字符串永远不会生成。

  因此，当字符串包含动态信息（例如 `string.Format` 或通过字符串插值表达式生成的字符串值）时，建议使用匿名委托通过 `=>` 表达式函数调用日志方法：

  ```csharp
  Logger.LogDebug(() => $"downstream templates are {string.Join(", ", response.Data.Route.DownstreamRoute.Select(r => r.DownstreamPathTemplate.Value))}");
  ```

  否则，常量字符串即可：

  ```csharp
  Logger.LogDebug("My const string");
  ```

**性能评估**

Ocelot 的日志记录性能在 22.0 版本中得到了改进（参见 PR 1745）。这些更改是应 Issue 1744 的要求，在团队讨论后提出的。

**顶级日志性能？**  
这里是你的生产环境的快速配方！你需要确保最低级别为 Critical 或 None。仅此而已！要确保顶级日志性能，意味着由日志提供程序写入的日志记录越少越好。因此，日志应该是相当空的。

不过，在第一次将版本发布到生产环境时，我们建议使用 Error 级别来观察系统和当前版本应用程序的行为。如果发布工程师确保生产环境中的版本稳定，则可以将最低级别提高到 Critical 或 None，以获得顶级性能。技术上，这将完全关闭日志记录功能。

**运行基准测试**

我们目前有两种类型的基准测试：

- **SerilogBenchmarks**：使用 Serilog 将日志记录到文件中。参见 `ConfigureLogging` 方法中的 `logging.AddSerilog(_logger)`。

- **MsLoggerBenchmarks**：使用 MS 默认日志记录到 MS Console 中。参见 `ConfigureLogging` 方法中的 `logging.AddConsole()`。

基准测试结果在很大程度上取决于运行环境和硬件。我们诚挚地邀请你按照以下说明在你的机器上运行日志基准测试。

1. 打开 PowerShell 或命令提示符控制台。
2. 以 Release 模式构建 Ocelot 解决方案：`dotnet build --configuration Release`
3. 转到 `test\Ocelot.Benchmarks\bin\Release\` 文件夹。
4. 选择 .NET 版本，切换到相应文件夹，例如 `net8.0`。
5. 运行 `Ocelot.Benchmarks.exe`：`.\Ocelot.Benchmarks.exe`
6. 运行 SerilogBenchmarks 或 MsLoggerBenchmarks，按下适当的基准测试编号：5 或 6，然后按 Enter 键。
7. 等待 3 分钟以上完成基准测试，并获取最终结果。
8. 阅读并分析你的基准测试会话结果。

**指标**  
待开发…

## 元数据

**配置**

Ocelot 提供了各种功能，如路由、认证、缓存、负载均衡等。然而，一些用户可能会遇到 Ocelot 无法满足其特定需求的情况，或者他们希望自定义其行为。在这种情况下，Ocelot 允许用户在路由配置中添加元数据。这一属性可以存储任何任意数据，用户可以在中间件或委托处理程序中访问这些数据。

通过使用元数据，用户可以实现自己的逻辑并扩展 Ocelot 的功能。例如：

```json
{
  "Routes": [
    {
      "UpstreamHttpMethod": ["GET"],
      "UpstreamPathTemplate": "/posts/{postId}",
      "DownstreamPathTemplate": "/api/posts/{postId}",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 80 }
      ],
      "Metadata": {
        "id": "FindPost",
        "tags": "tag1, tag2, area1, area2, func1",
        "plugin1.enabled": "true",
        "plugin1.values": "[1, 2, 3, 4, 5]",
        "plugin1.param": "value2",
        "plugin1.param2": "123",
        "plugin2/param1": "overwritten-value",
        "plugin2/param2": "{\"name\":\"John Doe\",\"age\":30,\"city\":\"New York\",\"is_student\":false,\"hobbies\":[\"reading\",\"hiking\",\"cooking\"]}"
      }
    }
  ],
  "GlobalConfiguration": {
    "Metadata": {
      "instance_name": "machine-1",
      "plugin2/param1": "default-value"
    }
  }
}
```

现在，可以通过 `DownstreamRoute` 对象访问路由元数据：

```csharp
public class MyMiddleware
{
    public Task Invoke(HttpContext context, Func<Task> next)
    {
        var route = context.Items.DownstreamRoute();
        var enabled = route.GetMetadata<bool>("plugin1.enabled");
        var values = route.GetMetadata<string[]>("plugin1.values");
        var param1 = route.GetMetadata<string>("plugin1.param", "system-default-value");
        var param2 = route.GetMetadata<int>("plugin1.param2");

        // 处理 plugin1 的功能

        return next?.Invoke();
    }
}
```

**扩展方法**

Ocelot 提供了一个 `DownstreamRoute` 扩展方法，以帮助你轻松地检索元数据值。除了 `string`、`bool`、`bool?`、`string[]` 和数值类型之外，所有传递的字符串参数都被视为 JSON 字符串，并尝试将其转换为泛型类型 `T` 的对象。如果值为 `null`，则返回所选目标类型的默认值（如果未明确指定）。

方法 | 描述 | 备注
--- | --- | ---
`GetMetadata<string>` | 元数据值作为字符串返回，无需进一步解析 | 
`GetMetadata<string[]>` | 元数据值按给定分隔符（默认 `,`）拆分，并返回为字符串数组 | 可以在全局配置中设置多个参数，如分隔符（默认 = `","`）、`StringSplitOptions`（默认 None）和 `TrimChars`（应删除的字符，默认 = `[' ']`）。
`GetMetadata<Any known numeric type>` | 元数据值被解析为数字 | 可以在全局配置中设置一些参数，如 `NumberStyle`（默认 Any）和 `CurrentCulture`（默认 `CultureInfo.CurrentCulture`）。
`GetMetadata<T>` | 元数据值转换为给定的泛型类型。值被视为 JSON 字符串，JSON 序列化器尝试将字符串反序列化为目标类型 | 可以将 `JsonSerializerOptions` 对象作为方法参数传递，默认使用 Web。
`GetMetadata<bool>` | 检查元数据值是否为真值，否则返回 `false` | 真值包括：`true`、`yes`、`ok`、`on`、`enable`、`enabled`。
`GetMetadata<bool?>` | 检查元数据值是否为真值（返回 `true`）或假值（返回 `false`），否则返回 `null` | 已知真值包括：`true`、`yes`、`ok`、`on`、`enable`、`enabled`、`1`。已知假值包括：`false`、`no`、`off`、`disable`、`disabled`、`0`。

## 方法转换

Ocelot 允许用户在向下游服务发出请求时更改 HTTP 请求方法。

这通过设置以下路由配置来实现：

```json
{
  "UpstreamPathTemplate": "/{url}",
  "DownstreamPathTemplate": "/{url}",
  "DownstreamScheme": "http",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 54321 }
  ],
  "UpstreamHttpMethod": ["Get"],
  "DownstreamHttpMethod": "POST" // !
}
```

这里的关键属性是 `DownstreamHttpMethod`，它被设置为 `POST`，而路由将仅在 `UpstreamHttpMethod` 设置的 `GET` 方法时匹配。

此功能在与仅支持 `POST` 的下游 API 交互时非常有用，特别是当你希望提供某种 RESTful 接口时。

## 中间件注入

警告：使用时请谨慎！如果你在中间件管道中看到任何异常或奇怪的行为，并且你使用了以下中间件，请将其移除并再试一次！

在你的 `Startup.cs` 中设置 Ocelot 时，你可以提供一些额外的中间件并覆盖默认中间件。这可以通过如下方式完成：

```csharp
var configuration = new OcelotPipelineConfiguration
{
    PreErrorResponderMiddleware = async (context, next) =>
    {
        await next.Invoke();
    }
};
app.UseOcelot(configuration);
```

在上面的示例中，提供的函数会在第一个 Ocelot 中间件之前运行。这允许用户在 Ocelot 管道运行之前和之后提供任何他们想要的行为。这意味着你可以破坏所有内容，所以请随意使用！

用户可以对以下中间件进行设置（详见 OcelotPipelineConfiguration 类）：

- **PreErrorResponderMiddleware**: 上述已经解释。
- **PreAuthenticationMiddleware**: 允许用户在调用 Ocelot 身份验证中间件之前运行身份验证逻辑。
- **AuthenticationMiddleware**: 覆盖 Ocelot 的身份验证中间件。[1]
- **PreAuthorizationMiddleware**: 允许用户在调用 Ocelot 授权中间件之前运行授权逻辑。
- **AuthorizationMiddleware**: 覆盖 Ocelot 的授权中间件。[1]
- **PreQueryStringBuilderMiddleware**: 允许用户在请求被传递给 Ocelot 请求创建器之前操控查询字符串。

显然，你可以像正常一样在调用 `app.UseOcelot()` 之前添加提到的 Ocelot 中间件覆盖。它不能在调用之后添加，因为 Ocelot 不会根据指定的中间件配置调用下一个 Ocelot 中间件覆盖。因此，接下来的中间件将不会影响 Ocelot 配置。

**ASP.NET Core 中间件与 Ocelot 管道构建器**

Ocelot 管道是整个 ASP.NET Core 中间件链（即应用程序管道）的一部分。`BuildOcelotPipeline` 方法封装了 Ocelot 管道。`BuildOcelotPipeline` 方法中的最后一个中间件是 `HttpRequesterMiddleware`，它会调用下一个中间件（如果添加到管道中）。

内部 `HttpRequesterMiddleware` 是管道的一部分，但它是私有的，不能被覆盖，因为这个中间件不属于用户可以覆盖的公共中间件列表！因此，这是整个 Ocelot 和 ASP.NET 管道的最后一个中间件，处理非用户操作。最后一个用户（公共）中间件是 `PreQueryStringBuilderMiddleware`，它从管道配置对象中读取，详见前述部分。

考虑到 `PreQueryStringBuilderMiddleware` 和 `HttpRequesterMiddleware` 是最后的用户和系统中间件，管道中没有其他中间件。但是，你仍然可以扩展 ASP.NET 管道，如下代码所示：

```csharp
app.UseOcelot().Wait();
app.UseMiddleware<MyCustomMiddleware>();
```

但我们不推荐在调用 `UseOcelot()` 之前或之后添加自定义中间件，因为这会影响整个管道的稳定性，并且没有经过测试。这种类型的自定义管道构建超出了 Ocelot 管道模型，其解决方案的质量由你自行承担风险。

最后，不要对系统（私有，不能被覆盖）和用户（公共，可被覆盖）中间件之间的区别感到困惑。私有中间件是隐藏的，不能被覆盖，但整个 ASP.NET 管道仍然可以扩展。公共中间件是完全可定制的，可以被覆盖。

**未来**

社区对添加更多可覆盖中间件表现出了兴趣。其中一个请求是 PR 1497，可能会被包括在下一个发布版本中。

无论如何，如果你认为当前的可覆盖中间件没有提供足够的管道灵活性，你可以在仓库的讨论空间中提出新的话题。 🐙

[1] 请谨慎使用提到的中间件覆盖！覆盖中间件会移除默认实现！如果你在中间件管道中看到任何异常或奇怪的行为，请移除覆盖的中间件并再试一次！

## 服务质量

**标签:** QoS

Ocelot 当前支持一个 QoS 功能。它允许你为每个路由配置使用断路器（circuit breaker）以管理对下游服务的请求。此功能利用了一个出色的 .NET 库，名为 Polly。有关更多信息，请访问 Polly 的官方仓库。

### 安装

要使用管理 API，首先需要导入相关的 NuGet 包：

```powershell
Install-Package Ocelot.Provider.Polly
```

接下来，在 `ConfigureServices` 方法中，通过在 `AddOcelot()` 返回的 `OcelotBuilder` 上调用 `AddPolly()` 扩展方法来集成 Polly 服务，如下所示：

```csharp
services.AddOcelot()
    .AddPolly();
```

### 配置

然后在路由配置中添加以下部分：

```json
"QoSOptions": {
  "ExceptionsAllowedBeforeBreaking": 3,
  "DurationOfBreak": 1000,
  "TimeoutValue": 5000
}
```

你必须将 `ExceptionsAllowedBeforeBreaking` 设置为 2 或更大的值，才能实现此规则。

- **DurationOfBreak** 指定断路器在触发后将保持打开状态 1 秒。
- **TimeoutValue** 指定如果请求超过 5 秒，将自动超时。

### 断路器策略

`ExceptionsAllowedBeforeBreaking` 和 `DurationOfBreak` 可以独立于 `TimeoutValue` 进行配置：

```json
"QoSOptions": {
  "ExceptionsAllowedBeforeBreaking": 3,
  "DurationOfBreak": 1000
}
```

另外，你可以省略 `DurationOfBreak`，默认为 Polly 文档中的 5 秒：

```json
"QoSOptions": {
  "ExceptionsAllowedBeforeBreaking": 3
}
```

此设置只激活断路器策略。

### 超时策略

`TimeoutValue` 可以独立于 `ExceptionsAllowedBeforeBreaking` 和 `DurationOfBreak` 设置：

```json
"QoSOptions": {
  "TimeoutValue": 5000
}
```

此设置只激活超时策略。

### 注意事项

- 如果没有 QoS 部分，QoS 将不会被使用，Ocelot 将对所有下游请求施加默认的 90 秒超时。要请求可配置性，请提交一个问题。
- 从版本 23.2 开始，不再支持 Polly V7 语法。
- 对于 Polly 版本 8 及以上，文档中规定了以下值的约束：
  - `ExceptionsAllowedBeforeBreaking` 值必须为 2 或更高。
  - `DurationOfBreak` 值必须超过 500 毫秒，默认为 5000 毫秒（5 秒），如果未指定或值为 500 毫秒或更少。
  - `TimeoutValue` 必须超过 10 毫秒。

请参考弹性策略文档以详细了解每个选项。

### 可扩展性

如果你想使用自定义的 `ResiliencePipeline<T>` 提供者，可以使用以下语法：

```csharp
services.AddOcelot()
    .AddPolly<MyProvider>();
// MyProvider 应实现 IPollyQoSResiliencePipelineProvider<HttpResponseMessage>
// 注意：你可以使用标准提供者 PollyQoSResiliencePipelineProvider
```

如果你还想使用自定义的 `DelegatingHandler`，可以使用以下语法：

```csharp
services.AddOcelot()
    .AddPolly<MyProvider>(MyQosDelegatingHandlerDelegate);
// MyProvider 应实现 IPollyQoSResiliencePipelineProvider<HttpResponseMessage>
// 注意：你可以使用标准提供者 PollyQoSResiliencePipelineProvider
// MyQosDelegatingHandlerDelegate 是用于获取 DelegatingHandler 的委托
```

最后，如果你想定义自己的异常映射集，可以使用以下语法：

```csharp
services.AddOcelot()
    .AddPolly<MyProvider>(MyErrorMapping);
// MyProvider 应实现 IPollyQoSResiliencePipelineProvider<HttpResponseMessage>
// 注意：你可以使用标准提供者 PollyQoSResiliencePipelineProvider

// MyErrorMapping 是一个 Dictionary<Type, Func<Exception, Error>>，例如：
private static readonly Dictionary<Type, Func<Exception, Error>> MyErrorMapping = new()
{
    {typeof(TaskCanceledException), CreateError},
    {typeof(TimeoutRejectedException), CreateError},
    {typeof(BrokenCircuitException), CreateError},
    {typeof(BrokenCircuitException<HttpResponseMessage>), CreateError},
};
private static Error CreateError(Exception e) => new RequestTimedOutError(e);
```

[1] `AddOcelot` 方法将默认的 ASP.NET 服务添加到 DI 容器中。你可以在配置服务时调用其他扩展的 `AddOcelotUsingBuilder` 方法以开发自定义构建器。请参阅“AddOcelotUsingBuilder 方法”部分的依赖注入功能中的更多说明。

[2] 如果有问题或遇到困难，请检查当前 QoS 问题，并按 QoS_label 标签进行筛选。

[3] 我们将 Polly 版本从 v7.x 升级到 v8.x！扩展性 3 功能是在问题 1875 中请求的，并由 PR 1914 作为版本 23.2 的一部分交付。


## 限流 (Rate Limiting)

**限流是什么？**
- [限流 - Wikipedia](https://en.wikipedia.org/wiki/Rate_limiting)
- [限流模式 - Azure 架构中心 | Microsoft Learn](https://learn.microsoft.com/en-us/azure/architecture/patterns/rate-limiting)
- [Google 限流](https://www.google.com/search?q=Rate+Limiting)

### Ocelot 自定义实现

Ocelot 提供了限流功能，以防止下游服务过载。以下是如何配置 Ocelot 的限流功能：

#### 按客户端头进行限流

要为路由实现限流，你需要在配置文件中包含以下 JSON 配置：

```json
"RateLimitOptions": {
  "ClientWhitelist": [], // 白名单客户端数组
  "EnableRateLimiting": true,
  "Period": "1s", // 秒、分钟、小时、天
  "PeriodTimespan": 1, // 仅限秒
  "Limit": 1
}
```

- **ClientWhitelist**: 包含被白名单列出的客户端数组。这些客户端将免于限流。有关 `ClientIdHeader` 选项的更多信息，请参见全局配置部分。
- **EnableRateLimiting**: 该设置启用端点的限流。
- **Period**: 该参数定义了限流适用的时间段，例如 `1s`（秒）、`5m`（分钟）、`1h`（小时）和 `1d`（天）。如果达到请求的确切限制，则立即发生超额，`PeriodTimespan` 开始计算。必须等待 `PeriodTimespan` 时长结束后才能再次发起请求。如果在该时间段内超过了允许的请求次数，将出现 `QuotaExceededMessage`，并附带一个 `HttpStatusCode`。
- **PeriodTimespan**: 该参数表示允许重试的时间（以秒为单位）。在此期间，响应中会出现 `QuotaExceededMessage` 和 `HttpStatusCode`。客户端应查阅 `Retry-After` 头以确定后续请求的时间。
- **Limit**: 该参数定义了客户端在指定时间段内允许发起的最大请求次数。

#### 全局配置

你可以在 `ocelot.json` 的 `GlobalConfiguration` 部分设置以下内容：

```json
"GlobalConfiguration": {
  "BaseUrl": "https://api.mybusiness.com",
  "RateLimitOptions": {
    "DisableRateLimitHeaders": false,
    "QuotaExceededMessage": "Customize Tips!",
    "HttpStatusCode": 418, // I'm a teapot
    "ClientIdHeader": "MyRateLimiting"
  }
}
```

- **DisableRateLimitHeaders**: 确定是否禁用 `X-Rate-Limit` 和 `Retry-After` 头。
- **QuotaExceededMessage**: 定义当超出配额时显示的消息。此项为可选，默认消息为信息性消息。
- **HttpStatusCode**: 指定在限流期间返回的 HTTP 状态码。默认值为 429（Too Many Requests）。
- **ClientIdHeader**: 指定用于识别客户端的头，默认为 `ClientId`。

#### 未来与 ASP.NET Core 实现

Ocelot 团队正在考虑重新设计限流功能，参考了 Brennan Conroy 于 2022 年 7 月 13 日发布的《Announcing Rate Limiting for .NET》。目前尚未做出决定，旧版功能仍然是 .NET 7 的 20.0 版本的一部分。

发现 ASP.NET Core 7.0 发布中的新功能：

- **RateLimiter 类**，自 ASP.NET Core 7.0 起可用
- **System.Threading.RateLimiting NuGet 包**
- **ASP.NET Core 的限流中间件文章**，作者包括 Arvin Kahbazi、Maarten Balliauw 和 Rick Anderson

虽然保留旧版实现作为 Ocelot 内置功能是有意义的，但我们计划过渡到 Microsoft.AspNetCore.RateLimiting 命名空间中的新 Rate Limiter。

请在我们仓库的讨论空间中分享你的想法。

[1] 历史上，“Ocelot 自定义限流”功能是 Ocelot 最古老和最早的功能之一。该功能在 GitHub 上由 @geffzhang 提交的 PR 37 中交付。非常感谢！它最初在版本 1.3.2 中发布。作者受到 @catcherwong 文章的启发撰写了此文档。

[2] 自 PR 37 和版本 1.3.2 以来，Ocelot 团队已审查并重新设计了该功能，以提供稳定的行为。修复 bug 1590（PR 1592）作为版本 23.3 的一部分发布。

## 请求聚合 (Request Aggregation)

Ocelot 提供了请求聚合功能，可以将多个常规路由组合起来，并将其响应映射为一个对象。这在客户端向服务器发起多个请求时，通常可以将其合并为一个请求。该功能允许你实现前端后台 (BFF) 类型的架构。

#### 基本 JSON 配置示例

```json
{
  "Routes": [
    {
      "UpstreamHttpMethod": [ "Get" ],
      "UpstreamPathTemplate": "/laura",
      "DownstreamPathTemplate": "/",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 51881 }
      ],
      "Key": "Laura"
    },
    {
      "UpstreamHttpMethod": [ "Get" ],
      "UpstreamPathTemplate": "/tom",
      "DownstreamPathTemplate": "/",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 51882 }
      ],
      "Key": "Tom"
    }
  ],
  "Aggregates": [
    {
      "UpstreamPathTemplate": "/",
      "RouteKeys": [ "Tom", "Laura" ]
    }
  ]
}
```

- **Routes**: 定义了两个正常的路由，每个路由都有一个 `Key` 属性。
- **Aggregates**: 定义了一个聚合，将两个路由通过 `RouteKeys` 组合起来，并指定了一个 `UpstreamPathTemplate`。

如果 `/tom` 路径返回 `{"Age": 19}`，`/laura` 路径返回 `{"Age": 25}`，则聚合后的响应将是：

```json
{
  "Tom": {"Age": 19},
  "Laura": {"Age": 25}
}
```

**注意**:
- 目前的聚合功能非常简单，Ocelot 仅将来自下游服务的响应放入 JSON 字典中。
- 所有头信息将从下游服务的响应中丢失。
- Ocelot 总是返回 `application/json` 内容类型。
- 如果下游服务返回 404 Not Found，聚合将不会改变响应为 404，即使所有下游服务都返回 404。

#### 复杂聚合

如果你需要使用聚合查询，但不知道所有查询参数，可以先调用一个端点来获取必要的数据（例如用户 ID），然后返回用户详细信息。

例如：
- `/Comments` 返回包含 `authorId` 属性的评论列表。
- `/users/{userId}` 通过将 `userId` 替换为 `authorId` 来获取用户详细信息。

这可以通过以下方式实现：

```csharp
new AggregateRouteConfig
{
    RouteKey = "UserDetails",
    JsonPath = "$[*].authorId",
    Parameter = "userId"
};
```

- **RouteKey**: 用作路由的参考。
- **JsonPath**: 指示你感兴趣的参数在第一次请求响应体中的位置。
- **Parameter**: 表示将 `authorId` 的值用于请求参数 `userId`。

#### 注册自定义聚合器

Ocelot 允许用户实现自定义聚合器，以处理来自下游服务的响应并将其聚合为响应对象。你可以在 `ocelot.json` 中添加 `Aggregator` 属性，如下所示：

```json
{
  "Routes": [
    {
      "UpstreamHttpMethod": [ "Get" ],
      "UpstreamPathTemplate": "/laura",
      "DownstreamPathTemplate": "/",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 51881 }
      ],
      "Key": "Laura"
    },
    {
      "UpstreamHttpMethod": [ "Get" ],
      "UpstreamPathTemplate": "/tom",
      "DownstreamPathTemplate": "/",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "localhost", "Port": 51882 }
      ],
      "Key": "Tom"
    }
  ],
  "Aggregates": [
    {
      "UpstreamPathTemplate": "/",
      "RouteKeys": [ "Tom", "Laura" ],
      "Aggregator": "FakeDefinedAggregator"
    }
  ]
}
```

**注册自定义聚合器示例**:

```csharp
services
    .AddOcelot()
    .AddSingletonDefinedAggregator<FakeDefinedAggregator>();
```

**自定义聚合器实现示例**:

```csharp
public class FakeDefinedAggregator : IDefinedAggregator
{
    public async Task<DownstreamResponse> Aggregate(List<HttpContext> responseHttpContexts)
    {
        var responses = responseHttpContexts.Select(x => x.Items.DownstreamResponse()).ToArray();

        var contentList = new List<string>();
        foreach (var response in responses)
        {
            var content = await response.Content.ReadAsStringAsync();
            contentList.Add(content);
        }

        return new DownstreamResponse(
            new StringContent(JsonConvert.SerializeObject(contentList)),
            HttpStatusCode.OK,
            responses.SelectMany(x => x.Headers).ToList(),
            "reason");
    }
}
```

#### 注意事项

- 无法使用具有特定 `RequestIdKeys` 的路由，因为跟踪这些请求将变得非常复杂。
- 聚合仅支持 GET HTTP 方法。
- 聚合允许将 `HttpRequest.Body` 转发到下游服务，但必须指定 `Content-Length` 头；否则，Ocelot 将记录警告。

[1] 该功能最初在 PR 79 中请求，并在 PR 298 中进行了进一步改进。2024 年 3 月 4 日，版本 23.1 对 Multiplexer 设计进行了重大重构和修订，请参见 PRs 1826 和 1462。

[2] `AddOcelot` 方法将默认的 ASP.NET 服务添加到依赖注入容器中。你可以在配置服务时调用另一个扩展的 `AddOcelotUsingBuilder` 方法，以开发自己的自定义构建器。有关更多说明，请参阅“AddOcelotUsingBuilder 方法”部分的依赖注入特性。

## 请求 ID
也称为关联 ID 或 HttpContext.TraceIdentifier

Ocelot 支持客户端以头部形式发送请求 ID。如果设置了，Ocelot 会在中间件管道中一旦请求 ID 可用时立即使用它进行日志记录。Ocelot 还会将请求 ID 与指定的头部一起转发到下游服务。

如果在日志配置中将 `IncludeScopes` 设置为 `true`，你仍然可以在日志中获取 ASP.NET Core 请求 ID。

要使用请求 ID 功能，你有两个选项。

#### 全局配置
在你的 `ocelot.json` 文件中的 `GlobalConfiguration` 部分设置以下内容。这将应用于所有进入 Ocelot 的请求。

```json
"GlobalConfiguration": {
  "RequestIdKey": "OcRequestId"
}
```

我们建议使用全局配置，除非你确实需要将其限制为特定的路由。

#### 路由特定配置
如果你希望为特定路由覆盖此设置，可以在 `ocelot.json` 中为该特定路由添加以下内容：

```json
"RequestIdKey": "OcRequestId"
```

一旦 Ocelot 确定了与路由对象匹配的传入请求，它将根据路由配置设置请求 ID。

#### 注意事项
这可能会导致一个小问题。如果设置了全局配置，可能会出现一个请求 ID 直到路由被识别，然后另一个请求 ID，因为请求 ID 键可能会更改。这是设计使然，也是目前我们认为的最佳解决方案。在这种情况下，OcelotLogger 将在日志中显示请求 ID 和之前的请求 ID。

以下是调试级别设置的正常请求日志示例：

```plaintext
dbug: Ocelot.Errors.Middleware.ExceptionHandlerMiddleware[0]
      requestId: asdf, previousRequestId: no previous request id, message: ocelot pipeline started,
dbug: Ocelot.DownstreamRouteFinder.Middleware.DownstreamRouteFinderMiddleware[0]
      requestId: asdf, previousRequestId: no previous request id, message: upstream url path is {upstreamUrlPath},
dbug: Ocelot.DownstreamRouteFinder.Middleware.DownstreamRouteFinderMiddleware[0]
      requestId: asdf, previousRequestId: no previous request id, message: downstream template is {downstreamRoute.Data.Route.DownstreamPath},
dbug: Ocelot.RateLimit.Middleware.ClientRateLimitMiddleware[0]
      requestId: asdf, previousRequestId: no previous request id, message: EndpointRateLimiting is not enabled for Ocelot.Values.PathTemplate,
dbug: Ocelot.Authorization.Middleware.AuthorizationMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: /posts/{postId} route does not require user to be authorized,
dbug: Ocelot.DownstreamUrlCreator.Middleware.DownstreamUrlCreatorMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: downstream url is {downstreamUrl.Data.Value},
dbug: Ocelot.Request.Middleware.HttpRequestBuilderMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: setting upstream request,
dbug: Ocelot.Requester.Middleware.HttpRequesterMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: setting http response message,
dbug: Ocelot.Responder.Middleware.ResponderMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: no pipeline errors, setting and returning completed response,
dbug: Ocelot.Errors.Middleware.ExceptionHandlerMiddleware[0]
      requestId: 1234, previousRequestId: asdf, message: ocelot pipeline finished,
```

更实际的生产环境（例如瑞士）示例如下：

```plaintext
warn: Ocelot.DownstreamRouteFinder.Middleware.DownstreamRouteFinderMiddleware[0]
      requestId: 0HMVD33IIJRFR:00000001, previousRequestId: no previous request id, message: DownstreamRouteFinderMiddleware setting pipeline errors. IDownstreamRouteFinder returned Error Code: UnableToFindDownstreamRouteError Message: Failed to match Route configuration for upstream path: /, verb: GET.
warn: Ocelot.Responder.Middleware.ResponderMiddleware[0]
      requestId: 0HMVD33IIJRFR:00000001, previousRequestId: no previous request id, message: Error Code: UnableToFindDownstreamRouteError Message: Failed to match Route configuration for upstream path: /, verb: GET. errors found in ResponderMiddleware. Setting error response for request path:/, request method: GET
```

#### 好奇吗？
请求 ID 是大型日志功能的一部分。

每个日志记录都有以下两个属性：

- `RequestId`：表示当前请求的 ID，例如 `0HMVD33IIJRFR:00000001`
- `PreviousRequestId`：表示上一个请求的 ID

作为注入到服务类构造函数中的 `IOcelotLogger` 接口对象，当前默认的 Ocelot 日志记录器（`OcelotLogger` 类）从 `IRequestScopedDataRepository` 接口对象中读取这两个属性。

## 路由

Ocelot 的主要功能是接受传入的 HTTP 请求并将其转发到下游服务。Ocelot 目前仅支持以另一种 HTTP 请求的形式进行转发（未来可能支持任何传输机制）。

Ocelot 将一个请求的路由描述为一个 Route。为了使 Ocelot 正常工作，你需要在配置中设置一个 Route。

```json
{
  "Routes": []
}
```

要配置一个 Route，你需要在 Routes JSON 数组中添加一个：

```json
{
  "UpstreamHttpMethod": [ "Put", "Delete" ],
  "UpstreamPathTemplate": "/posts/{postId}",
  "DownstreamPathTemplate": "/api/posts/{postId}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 80 }
  ]
}
```

- **DownstreamPathTemplate**、**DownstreamScheme** 和 **DownstreamHostAndPorts** 定义了请求将被转发到的 URL。

- **DownstreamHostAndPorts** 属性是一个集合，定义了你希望转发请求的下游服务的主机和端口。通常，这只包含一个条目，但有时你可能希望对下游服务进行负载均衡，Ocelot 允许你添加多个条目并选择负载均衡器。

- **UpstreamPathTemplate** 属性是 Ocelot 用来识别哪个 **DownstreamPathTemplate** 应用于给定请求的 URL。**UpstreamHttpMethod** 用于区分对同一 URL 的不同 HTTP 方法的请求。你可以设置特定的 HTTP 方法列表，也可以设置为空列表以允许任何方法。

#### 占位符

在 Ocelot 中，你可以在模板中添加变量占位符，格式为 `{something}`。占位符变量需要在 **DownstreamPathTemplate** 和 **UpstreamPathTemplate** 属性中都存在。当存在时，Ocelot 将尝试将 **UpstreamPathTemplate** 占位符中的值替换到 **DownstreamPathTemplate** 中。

你还可以使用 Catch All 类型的路由，例如：

```json
{
  "UpstreamHttpMethod": [ "Get", "Post" ],
  "UpstreamPathTemplate": "/{everything}",
  "DownstreamPathTemplate": "/api/{everything}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 80 }
  ]
}
```

这将转发任何路径 + 查询字符串组合到下游服务的 `/api` 路径。

注意，默认路由配置是不区分大小写的！

要更改此设置，你可以在每个路由的基础上指定以下设置：

```json
"RouteIsCaseSensitive": true
```

这意味着当 Ocelot 尝试将传入的上游 URL 与上游模板匹配时，评估将区分大小写。

#### 空占位符

这是占位符的一个特殊边界情况，其中占位符的值只是一个空字符串 `""`。

例如，给定一个路由：

```json
{
  "UpstreamPathTemplate": "/invoices/{url}",
  "DownstreamPathTemplate": "/api/invoices/{url}"
}
```

当 `{url}` 被指定时，它正常工作：`/invoices/123` → `/api/invoices/123`。

对于空占位符值，还有两个边界情况：
- 当 `{url}` 为空时，上游路径 `/invoices/` 应路由到下游路径 `/api/invoices/`。
- 省略最后的斜杠时，我们也期望上游路径 `/invoices` 被路由到下游路径 `/api/invoices`，这对人类来说是直观的。

#### Catch All

Ocelot 的路由还支持 Catch All 样式的路由，用户可以指定他们希望匹配所有流量。

如果你按以下配置设置，所有请求将被直接代理。占位符 `{url}` 名称不重要，任何名称都有效。

```json
{
  "UpstreamHttpMethod": [ "Get" ],
  "UpstreamPathTemplate": "/{url}",
  "DownstreamPathTemplate": "/{url}",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 80 }
  ]
}
```

Catch All 的优先级低于任何其他路由。如果你在配置中还有以下路由，Ocelot 会在 Catch All 之前匹配它：

```json
{
  "UpstreamHttpMethod": [ "Get" ],
  "UpstreamPathTemplate": "/",
  "DownstreamPathTemplate": "/",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "10.0.10.1", "Port": 80 }
  ]
}
```

#### 上游主机

此功能允许你根据上游主机来定义路由。它通过查看客户端使用的 Host 头部，然后将其作为识别路由的部分信息来工作。

要使用此功能，请在配置中添加以下内容：

```json
{
  "UpstreamHost": "somedomain.com"
}
```

上面的路由只有在 Host 头部值为 `somedomain.com` 时才会匹配。

如果你没有在路由中设置 **UpstreamHost**，那么任何 Host 头部都会匹配它。这意味着如果你有两个路由，它们除了 **UpstreamHost** 外完全相同，其中一个为空，另一个设置了 Ocelot 将优先选择已设置的路由。

#### 上游头部

除了按 **UpstreamPathTemplate** 路由外，你还可以定义 **UpstreamHeaderTemplates**。要匹配路由，请求头中必须包含字典对象中指定的所有头部。

```json
{
  "UpstreamPathTemplate": "/",
  "UpstreamHttpMethod": [ "Get" ],
  "UpstreamHeaderTemplates": {
    "country": "uk",
    "version": "v1"
  }
}
```

在这种情况下，只有在请求中包含两个指定的头部时，路由才会匹配。

##### 头部占位符

让我们探索一个更有趣的场景，在 **UpstreamHeaderTemplates** 中有效地使用占位符。

考虑以下使用特殊占位符格式 `{header:placeholdername}` 的方法：

```json
{
  "DownstreamPathTemplate": "/{versionnumber}/api",
  "DownstreamScheme": "https",
  "DownstreamHostAndPorts": [
    { "Host": "10.0.10.1", "Port": 80 }
  ],
  "UpstreamPathTemplate": "/api",
  "UpstreamHttpMethod": [ "Get" ],
  "UpstreamHeaderTemplates": {
    "version": "{header:versionnumber}"
  }
}
```

在这种情况下，请求头部 “version” 的整个值被插入到 **DownstreamPathTemplate** 中。如果需要，可以指定更复杂的上游头部模板，使用占位符如 `version-{header:version}_country-{header:country}`。

注意 1：**DownstreamPathTemplate** 中不需要占位符。此场景可以用于强制要求特定头部，而不考虑其值。

注意 2：此外，**UpstreamHeaderTemplates** 字典选项也适用于请求聚合。

#### 优先级

你可以通过在 `ocelot.json` 中包含 **Priority** 属性来定义你希望路由匹配 **Upstream HttpRequest** 的顺序。参见 issue 270 以了解更多信息。

```json
{
  "Priority": 0
}
```

0 是最低优先级，Ocelot 总是将 0 用于 `/{catchAll}` 路由，这仍然是硬编码的。在此之后，你可以设置任何你希望的优先级。

例如，你可以有：

```json
{
  "UpstreamPathTemplate": "/goods/{catchAll}",
  "Priority": 0
}
```

和

```json
{
  "UpstreamPathTemplate": "/goods/delete",
  "Priority": 1
}
```

在上面的示例中，如果你对 `/goods/delete` 进行请求，Ocelot 将匹配 `/goods/delete` 路由。以前，它会匹配 `/goods/{catchAll}`，因为这是列表中的第一个路由！

#### 查询字符串占位符

除了 URL 路径占位符，Ocelot 还能够转发查询字符串参数，格式为 `{something}`。此外，查询参数占位符需要在 **DownstreamPathTemplate** 和 **UpstreamPathTemplate** 属性中都存在。占位符替换在路径和查询字符串之间双向进行，但使用上有一些限制（参见查询参数合并）。

##### 从路径到查询字符串

Ocelot 允许你在 **DownstreamPathTemplate** 中指定查询字符串，例如：

```json
{
  "UpstreamPathTemplate": "/api/units/{subscription}/{unit}/updates",
  "DownstreamPathTemplate": "/api/subscriptions/{subscription}/updates?unitId={unit}"
}
```

在这个例子中，Ocelot 会使用上游路径模板中的 `{unit}` 占位符的值，并将

其添加到下游请求的查询字符串中，作为 `unitId` 参数！

注意！由于查询参数合并，最好将占位符名称与查询参数名称不同。

##### 从查询字符串到路径

Ocelot 还允许你在 **UpstreamPathTemplate** 中放置查询字符串参数，以便将特定查询匹配到特定服务：

```json
{
  "UpstreamPathTemplate": "/api/subscriptions/{subscriptionId}/updates?unitId={uid}",
  "DownstreamPathTemplate": "/api/units/{subscriptionId}/{uid}/updates"
}
```

在这个例子中，Ocelot 只会匹配具有匹配 URL 路径的请求，并且查询字符串以 `unitId=something` 开始。你可以有其他查询，但必须以匹配的参数开始。此外，Ocelot 将从查询字符串中交换 `{uid}` 参数，并将其用于下游请求路径。

注意，最佳实践是给予不同的占位符名称，而不是查询参数名称，因为查询参数合并可能会引发问题。

##### Catch All 查询字符串

Ocelot 的路由还支持 Catch All 样式的路由，以转发所有查询字符串参数。占位符 `{everything}` 名称不重要，任何名称都有效。

```json
{
  "UpstreamPathTemplate": "/contracts?{everything}",
  "DownstreamPathTemplate": "/apipath/contracts?{everything}"
}
```

这个完整的查询字符串路由功能在查询字符串不应该被转换而是直接路由的情况下非常有用，例如 OData 过滤器等（参见 issue 1174）。

注意，`{everything}` 占位符可以为空，同时捕获所有查询字符串，因为这是空占位符 1 功能的一部分！因此，上游路径 `/contracts?` 和 `/contracts` 都被路由到没有查询字符串的下游路径 `/apipath/contracts`。

##### 查询参数合并

查询字符串参数是无序的并且会合并以创建最终的下游 URL。此过程对于 **DownstreamUrlCreatorMiddleware** 需要控制占位符替换和重复参数的合并。一个在 **UpstreamPathTemplate** 中首次出现的参数可能在最终的下游 URL 中占据不同的位置。此外，如果 **DownstreamPathTemplate** 包含查询参数在开头，它们在 **UpstreamPathTemplate** 中的位置是不确定的，除非明确指定。

在典型的场景中，合并算法通过以下步骤构建最终的下游 URL 查询字符串：
1. 采用 **DownstreamPathTemplate** 中初始定义的查询参数，并将它们放在开头，进行必要的占位符替换。
2. 将所有来自 Catch All 查询字符串的参数（由占位符 `{everything}` 表示）添加到第二个位置（在第 1 步明确指定的参数之后）。
3. 将任何剩余的替换占位符值作为参数值附加到字符串的末尾（如果存在）。

**ASP.NET API** 的数组参数绑定问题
由于参数合并，ASP.NET API 对数组的特殊模型绑定格式 `selectedCourses=1050&selectedCourses=2000` 不受支持。这个查询字符串将合并为 `selectedCourses=1050`，从而导致数组数据丢失。上游客户端生成正确的查询字符串对于数组模型是至关重要的，例如 `selectedCourses[0]=1050&selectedCourses[1]=2000`。有关数组模型绑定的详细信息，请参阅文档：`Bind arrays and string values from headers and query strings`。

**控制参数存在性**
需要注意的是，由于 **DownstreamUrlCreatorMiddleware** 的合并算法实现，查询字符串占位符受限于命名。但是，这也提供了灵活性，可以通过参数名称来管理最终下游 URL 中参数的存在性。

考虑以下两个开发场景：

1. 开发人员希望在替换占位符后保留参数（参见 issue 473）。这需要使用以下模板定义：

```json
{
  "UpstreamPathTemplate": "/path/{serverId}/{action}",
  "DownstreamPathTemplate": "/path2/{action}?server={serverId}"
}
```

在这里，`{serverId}` 占位符和 `server` 参数名称不同！最终，`server` 参数被保留。

重要的是要注意，由于名称的大小写敏感比较，`server` 参数将不会保留使用 `{server}` 占位符。然而，使用 `{Server}` 占位符是可以保留参数的。

2. 开发人员希望在替换占位符后删除过时的参数（参见 issue 952）。为此，必须使用具有大小写敏感比较的相同名称：

```json
{
  "UpstreamPathTemplate": "/users?userId={userId}",
  "DownstreamPathTemplate": "/persons?personId={userId}"
}
```

因此，`{userId}` 占位符和 `userId` 参数具有相同的名称！随后，`userId` 参数被删除。

需要注意的是，由于名称比较是大小写敏感的，如果使用 `{userid}` 占位符，则 `userId` 参数将不会被删除！

#### 安全选项

Ocelot 允许你使用 **IPAddressRange** 包管理允许/阻止的 IP 地址模式，使用 MPL-2.0 许可证。

此功能旨在允许更大的 IP 管理，以包含或排除通过 CIDR 表示法或 IP 范围的广泛 IP 范围。目前管理的模式如下：

- 单个 IP：`192.168.1.1`
- IP 范围：`192.168.1.1-192.168.1.250`
- IP 短范围：`192.168.1.1-250`
- IP 范围与子网：`192.168.1.0/255.255.255.0`
- CIDR：`192.168.1.0/24`
- IPv6 的 CIDR：`fe80::/10`

允许/阻止的列表在配置加载期间进行评估。

**ExcludeAllowedFromBlocked** 属性旨在提供指定广泛的阻止 IP 地址范围并允许 IP 地址子范围的能力。默认值：`false`。

如果 **SecurityOptions** 中没有该属性，则取默认值。

```json
{
  "SecurityOptions": {
    "IPBlockedList": [ "192.168.0.0/23" ],
    "IPAllowedList": ["192.168.0.15", "192.168.1.15"],
    "ExcludeAllowedFromBlocked": true
  }
}
```

#### 动态路由

动态路由的概念是当使用服务发现提供者时启用动态路由，这样你就不必提供 Route 配置。如果这听起来很有趣，请查看动态路由文档。

### 路由功能详细说明

#### 空占位符（Empty Placeholders）

- **功能介绍**：空占位符特性允许在 URL 路径中使用 `{}` 表示空值的情况。
- **版本信息**：自版本 23.0 起可用，具体细节请参阅 issue 748 和 23.0 版本发布说明。
  
#### 上游主机（Upstream Host）

- **功能介绍**：允许根据请求的 Host 头部来路由请求。
- **请求背景**：功能请求出现在 issue 216 中。

#### 上游头部（Upstream Headers）

- **功能介绍**：允许根据请求头部的值来路由请求。
- **功能提出**：在 issue 360 中提出，版本 24.0 中发布。

#### 安全选项（Security Options）

- **功能介绍**：提供了对 IP 地址范围的管理，包括允许和阻止特定 IP 地址的功能。
- **请求背景**：最初在 version 12.0.1 的 issue 628 中请求，随后在 issue 1400 中进行了重新设计和改进，并在版本 20.0 文档中发布。

#### 动态路由（Dynamic Routing）

- **功能介绍**：在使用服务发现提供者时支持动态路由。
- **请求背景**：功能请求出现在 issue 340 中。有关更多详细信息，请参见动态路由文档。

## 服务发现

Ocelot 允许您指定一个服务发现提供者，并将其用于查找 Ocelot 转发请求的下游服务的主机和端口。目前，这仅在 GlobalConfiguration 部分受支持，这意味着在路由级别指定 ServiceName 时，将为所有路由使用相同的服务发现提供者。

#### Consul
**命名空间:** Ocelot.Provider.Consul

首先，您需要安装 Ocelot.Provider.Consul 包，它提供了 Ocelot 对 Consul 的支持：

```bash
Install-Package Ocelot.Provider.Consul
```

要注册 Consul 服务，必须使用 AddOcelot() 返回的 OcelotBuilder 调用 AddConsul() 扩展方法。因此，在 ConfigureServices 方法中包含以下内容：

```csharp
services.AddOcelot()
    .AddConsul(); // 或者 .AddConsul<T>()
```

目前，有两种类型的 Consul 服务发现提供者：Consul 和 PollConsul。默认提供者是 Consul，这意味着如果 ConsulProviderFactory 无法读取、理解或解析 ServiceProviderConfiguration 对象的 Type 属性，则工厂将创建一个 Consul Provider 实例。

探索这些提供者类型并了解它们之间的区别，请参见子节：Consul Provider 和 PollConsul Provider。

#### KV Store 配置
在注册服务时，Ocelot 会尝试将其配置存储在 Consul KV Store 中。添加以下内容：

```csharp
services.AddOcelot()
    .AddConsul()
    .AddConfigStoredInConsul(); // !
```

您还需要在 ocelot.json 中添加以下内容。这是 Ocelot 查找 Consul 代理并与之交互以从 Consul 加载和存储配置的方式。

```json
"GlobalConfiguration": {
  "ServiceDiscoveryProvider": {
    "Host": "localhost",
    "Port": 9500
  }
}
```

团队在研究 Raft 共识算法并发现其非常困难后，决定创建此功能。为什么不利用 Consul 已经提供的功能呢？这意味着，如果您想充分利用 Ocelot，目前需要将 Consul 作为依赖项。

注意！此功能在向本地 Consul 代理发出新请求之前有 3 秒的 TTL 缓存。

#### Consul 配置键
如果您使用 Consul 进行配置（或将来使用其他提供者），您可能希望对配置进行键化，以便拥有多个配置。

为了指定键，您需要在配置 JSON 文件的 ServiceDiscoveryProvider 选项中设置 ConfigurationKey 属性，例如：

```json
"GlobalConfiguration": {
  "ServiceDiscoveryProvider": {
    "Host": "localhost",
    "Port": 9500,
    "ConfigurationKey": "Ocelot_A" // !
  }
}
```

在这个例子中，Ocelot 将使用 Ocelot_A 作为在 Consul 中查找配置的键。如果您未设置 ConfigurationKey，Ocelot 将使用字符串 InternalConfiguration 作为键。

#### Consul Provider
**类:** Ocelot.Provider.Consul.Consul

以下内容在 GlobalConfiguration 中是必需的。ServiceDiscoveryProvider 属性是必需的，如果您没有指定主机和端口，则将使用 Consul 的默认值。

请注意 Scheme 选项的默认值为 HTTP。它是在 PR 1154 中添加的。默认为 HTTP 以避免引入破坏性更改。

```json
"ServiceDiscoveryProvider": {
  "Scheme": "https",
  "Host": "localhost",
  "Port": 8500,
  "Type": "Consul"
}
```

将来我们可以添加允许路由特定配置的功能。

为了告诉 Ocelot 路由使用服务发现提供者获取主机和端口，您必须在请求下游时添加 ServiceName 和负载均衡器。目前，Ocelot 提供了 RoundRobin 和 LeastConnection 算法供您使用。如果未指定负载均衡器，Ocelot 将不会进行负载均衡。

```json
{
  "ServiceName": "product",
  "LoadBalancerOptions": {
    "Type": "LeastConnection"
  }
}
```

设置好后，Ocelot 将从服务发现提供者中查找下游主机和端口，并在可用服务之间进行负载均衡。

#### PollConsul Provider
**类:** Ocelot.Provider.Consul.PollConsul

许多人要求团队实现一个功能，使 Ocelot 可以轮询 Consul 获取最新的服务信息，而不是每个请求。如果您希望轮询 Consul 获取最新的服务而不是每个请求（默认行为），则需要设置以下配置：

```json
"ServiceDiscoveryProvider": {
  "Host": "localhost",
  "Port": 8500,
  "Type": "PollConsul",
  "PollingInterval": 100
}
```

轮询间隔以毫秒为单位，告诉 Ocelot 多久调用一次 Consul 以检查服务配置的变化。

请注意，这里有权衡。如果您轮询 Consul，Ocelot 可能会因为轮询间隔而不知道某个服务是否宕机，并且您可能会遇到比每个请求获取最新服务更多的错误。这实际上取决于您的服务的波动性。我们怀疑这对大多数人来说并不会产生太大影响，轮询可能会比每次请求调用 Consul（作为边车代理）带来微小的性能提升。如果您调用的是远程 Consul 代理，则轮询将是一个很好的性能提升。

#### 服务定义
您的服务需要添加到 Consul 中，如下所示（C# 风格，但希望这能有所帮助）……唯一需要注意的是不要在 Address 字段中添加 http 或 https。我们之前收到过关于不接受 Address 中的协议的反馈。经过阅读 Agents Overview 和 Define services 文档后，我们认为协议不应在其中。

在 C# 中：

```csharp
new AgentService()
{
    Service = "some-service-name",
    Address = "localhost",
    Port = 8080,
    ID = "some-id",
}
```

或者，在 JSON 中：

```json
"Service": {
  "ID": "some-id",
  "Service": "some-service-name",
  "Address": "localhost",
  "Port": 8080
}
```

#### ACL Token
如果您在 Consul 中使用 ACL，Ocelot 支持添加 X-Consul-Token 头。为了使其工作，您必须添加以下附加属性：

```json
"ServiceDiscoveryProvider": {
  "Host": "localhost",
  "Port": 8500,
  "Type": "Consul",
  "Token": "footoken"
}
```

Ocelot 将此令牌添加到用于发出请求的 Consul 客户端中，并且每个请求都将使用该令牌。

#### Consul 服务构建器
**接口:** IConsulServiceBuilder  
**实现:** DefaultConsulServiceBuilder

Ocelot 社区一致报告，过去和现在，由于各种 Consul 代理定义，Consul 服务（如连接性）存在问题。一些 DevOps 工程师更喜欢通过自定义主机名与节点名的分配来将服务分组为 Consul 目录节点，而其他人则专注于将代理服务定义为纯 IP 地址作为主机，这涉及到 954 bug 问题。

从版本 13.5.2 开始，在 PR 909 中构建服务下游主机/端口的逻辑被修改为优先考虑节点名称作为主机，而不是代理服务地址 IP。

版本 23.3 引入了一个自定义功能，通过 DefaultConsulServiceBuilder 类控制服务构建过程。该类具有虚拟方法，可以重写以满足开发人员和 DevOps 的需求。

DefaultConsulServiceBuilder 类中的当前逻辑如下：

```csharp
protected virtual string GetDownstreamHost(ServiceEntry entry, Node node)
    => node != null ? node.Name : entry.Service.Address;
```

一些 DevOps 工程师选择忽略节点名称，改为使用抽象标识符而不是实际的主机名。我们的团队则倡导将真实的主机名或 IP 地址分配给节点名称，将其视为最佳实践。如果这种方法不符合您的需求，或者如果您不想花时间为下游服务详细说明您的节点，您可能考虑定义没有节点名称的代理服务。在 Consul 设置中，您需要重写 DefaultConsulServiceBuilder 类的行为。有关更多详细信息，请参见下面的部分。

#### AddConsul<T> 方法
**签名:** `IOcelotBuilder AddConsul<TServiceBuilder>(this IOcelotBuilder builder)`

重写 DefaultConsulServiceBuilder 行为涉及两个步骤：定义一个新的类继承自 IConsulServiceBuilder 接口，然后使用 AddConsul<TServiceBuilder> 辅助方法将这种新行为注入到 DI 中。然而，最快且最简化的方法是直接继承 DefaultConsulServiceBuilder 类，它提供了更大的灵活性。

首先，我们需要定义一个新的服务构建类：

```csharp
public class MyConsulServiceBuilder : DefaultConsulServiceBuilder
{
    public MyConsulServiceBuilder(Func<ConsulRegistryConfiguration> configurationFactory, IConsulClientFactory clientFactory, IOcelotLoggerFactory loggerFactory)
        : base(configurationFactory, clientFactory

, loggerFactory) { }
    // 我想使用代理服务的 IP 地址作为下游主机名
    protected override string GetDownstreamHost(ServiceEntry entry, Node node) => entry.Service.Address;
}
```

其次，我们必须将新行为注入 DI，如下所示：

```csharp
services.AddOcelot()
    .AddConsul<MyConsulServiceBuilder>();
```

您可以参考存储库中的验收测试以获取示例。

### Eureka

此功能是作为问题 262 的一部分请求的，目的是为 Netflix Eureka 服务发现提供者添加支持。主要原因是 Eureka 是 Steeltoe 的关键组成部分，而 Steeltoe 是与 Pivotal 相关的！无论如何，背景说完了。

首先，您需要安装提供 Eureka 支持的 `Ocelot.Provider.Eureka` 包：

```bash
Install-Package Ocelot.Provider.Eureka
```

然后在 `ConfigureServices` 方法中添加以下内容：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddOcelot().AddEureka();
}
```

接着，在 `ocelot.json` 中添加以下内容：

```json
{
  "ServiceDiscoveryProvider": {
    "Type": "Eureka"
  }
}
```

根据此处的指南，您可能还需要在 `appsettings.json` 中添加一些内容。例如，下面的 JSON 告诉 Steeltoe / Pivotal 服务在哪里查找服务发现服务器，以及服务是否应注册：

```json
{
  "eureka": {
    "client": {
      "serviceUrl": "http://localhost:8761/eureka/",
      "shouldRegisterWithEureka": false,
      "shouldFetchRegistry": true
    }
  }
}
```

- 如果 `shouldRegisterWithEureka` 为 `false`，则 `shouldFetchRegistry` 默认为 `true`，因此您可以不显式设置它，但将其保留在这里。

Ocelot 将在启动时注册所有必要的服务，如果您有上述 JSON，它将注册自己到 Eureka。服务会每 30 秒（默认）轮询 Eureka，获取最新的服务状态并将其保存在内存中。当 Ocelot 请求特定服务时，从内存中检索，因此性能不会成为大问题。

Ocelot 将使用 Eureka 中设置的协议（http、https），如果这些值未在 `ocelot.json` 中提供。

### 动态路由

此功能是作为问题 340 的一部分请求的。其思想是，当使用服务发现提供者时启用动态路由（有关更多信息，请参见该文档部分）。在这种模式下，Ocelot 将使用上游路径的第一个段来查找下游服务。

例如，如果使用 URL `https://api.mywebsite.com/product/products` 调用 Ocelot，Ocelot 将取路径的第一个段 `product` 并将其用作键来查找 Consul 中的服务。如果 Consul 返回服务，Ocelot 将使用从 Consul 返回的主机和端口以及其余的路径段（此例中为 `products`）来请求下游服务，即 `http://hostfromconsul:portfromconsul/products`。Ocelot 将像往常一样将任何查询字符串附加到下游 URL。

请注意，要启用动态路由，您的配置中需要有 0 个路由。目前，您不能混合动态和配置路由。此外，您需要指定服务发现提供者的详细信息以及下游 http/https 协议作为 `DownstreamScheme`。

此外，您可以设置 `RateLimitOptions`、`QoSOptions`、`LoadBalancerOptions` 和 `HttpHandlerOptions`，这些选项将应用于所有动态路由。`DownstreamScheme` 允许您设置 Ocelot 在 https 上调用，但与私有服务通过 http 通信的情况。

配置可能如下所示：

```json
{
  "Routes": [],
  "Aggregates": [],
  "GlobalConfiguration": {
    "RequestIdKey": null,
    "ServiceDiscoveryProvider": {
      "Host": "localhost",
      "Port": 8500,
      "Type": "Consul",
      "Token": null,
      "ConfigurationKey": null
    },
    "RateLimitOptions": {
      "ClientIdHeader": "ClientId",
      "QuotaExceededMessage": null,
      "RateLimitCounterPrefix": "ocelot",
      "DisableRateLimitHeaders": false,
      "HttpStatusCode": 429
    },
    "QoSOptions": {
      "ExceptionsAllowedBeforeBreaking": 0,
      "DurationOfBreak": 0,
      "TimeoutValue": 0
    },
    "BaseUrl": null,
    "LoadBalancerOptions": {
      "Type": "LeastConnection",
      "Key": null,
      "Expiry": 0
    },
    "DownstreamScheme": "http",
    "HttpHandlerOptions": {
      "AllowAutoRedirect": false,
      "UseCookieContainer": false,
      "UseTracing": false
    }
  }
}
```

Ocelot 还允许您设置 `DynamicRoutes` 集合，这样可以为每个下游服务设置速率限制规则。如果您有例如产品和搜索服务，并且想要对一个服务进行更多的速率限制，可以参考以下示例：

```json
{
  "DynamicRoutes": [
    {
      "ServiceName": "product",
      "RateLimitRule": {
        "ClientWhitelist": [],
        "EnableRateLimiting": true,
        "Period": "1s",
        "PeriodTimespan": 1000.0,
        "Limit": 3
      }
    }
  ],
  "GlobalConfiguration": {
    "RequestIdKey": null,
    "ServiceDiscoveryProvider": {
      "Host": "localhost",
      "Port": 8523,
      "Type": "Consul"
    },
    "RateLimitOptions": {
      "ClientIdHeader": "ClientId",
      "QuotaExceededMessage": "",
      "RateLimitCounterPrefix": "",
      "DisableRateLimitHeaders": false,
      "HttpStatusCode": 428
    },
    "DownstreamScheme": "http"
  }
}
```

此配置意味着如果请求到达 Ocelot 路径 `/product/*`，则动态路由将生效，Ocelot 将使用 `DynamicRoutes` 部分中设置的速率限制规则对 `product` 服务进行限制。

请详细阅读所有文档以了解这些选项。

### 自定义提供者

Ocelot 还允许您创建自己的服务发现实现。这是通过实现 `IServiceDiscoveryProvider` 接口来完成的，如下所示：

```csharp
public class MyServiceDiscoveryProvider : IServiceDiscoveryProvider
{
    private readonly DownstreamRoute _downstreamRoute;

    public MyServiceDiscoveryProvider(DownstreamRoute downstreamRoute)
    {
        _downstreamRoute = downstreamRoute;
    }

    public async Task<List<Service>> Get()
    {
        var services = new List<Service>();
        //...
        // 将匹配 _downstreamRoute 的服务添加到列表中
        return services;
    }
}
```

在 `ocelot.json` 中将其类名设置为提供者类型：

```json
{
  "GlobalConfiguration": {
    "ServiceDiscoveryProvider": {
      "Type": "MyServiceDiscoveryProvider"
    }
  }
}
```

最后，在应用程序的 `ConfigureServices` 方法中，注册一个 `ServiceDiscoveryFinderDelegate` 以初始化并返回提供者：

```csharp
ServiceDiscoveryFinderDelegate serviceDiscoveryFinder = (provider, config, route) =>
{
    return new MyServiceDiscoveryProvider(route);
};
services.AddSingleton(serviceDiscoveryFinder);
services.AddOcelot();
```

### 自定义提供者示例

为了引入自定义服务发现提供者的基本模板，我们准备了一个示例：

- **链接**: `samples / OcelotServiceDiscovery`
- **解决方案**: `Ocelot.Samples.ServiceDiscovery.sln`

此解决方案包含以下项目：

- **ApiGateway**
- **DownstreamService**

该解决方案已准备好进行任何部署。所有服务都已绑定，意味着所有端口和主机都已准备好立即使用（在 Visual Studio 中运行）。

所有运行该解决方案的说明都在 `README.md` 中。

#### DownstreamService

该项目提供了一个可以在 ApiGateway 路由中重用的下游服务。它具有多个 `launchSettings.json` 配置文件，适用于您喜欢的启动和托管场景：Visual Studio 运行会话、Kestrel 控制台托管和 Docker 部署。

#### ApiGateway

该项目包含一个自定义服务发现提供者，并且在 `ocelot.json` 文件中仅包含对 DownstreamService 服务的路由。您可以添加更多路由！

自定义提供者的主要源代码在 `ServiceDiscovery` 文件夹中：`MyServiceDiscoveryProvider` 和 `MyServiceDiscoveryProviderFactory` 类。欢迎您设计和开发它们！

此外，该自定义提供者的基石是 `ConfigureServices` 方法，您可以在其中选择设计和实现选项：简单或更复杂：

```csharp
builder.ConfigureServices(s =>
{
    // 从应用程序配置进行初始化或硬编码/选择最佳选项。
    bool easyWay = true;

    if (easyWay)
    {
        // 设计 #1。定义一个自定义查找委托，以在默认工厂（即 ServiceDiscoveryProviderFactory）下实例化自定义提供者
        s.AddSingleton<ServiceDiscoveryFinderDelegate>((serviceProvider, config, downstreamRoute)
            => new MyServiceDiscoveryProvider(serviceProvider, config, downstreamRoute));
    }
    else
    {
        // 设计 #2。抽象出默认工厂（ServiceDiscoveryProviderFactory）和查找委托，
        // 通过实现 IServiceDiscoveryProviderFactory 接口来创建自己的工厂。
        s.RemoveAll<IServiceDiscoveryProviderFactory>();
        s.AddSingleton

<IServiceDiscoveryProviderFactory, MyServiceDiscoveryProviderFactory>();

        // 它不会被调用，但对内部验证是必要的，这也是一种黑客方式
        s.AddSingleton<ServiceDiscoveryFinderDelegate>((serviceProvider, config, downstreamRoute) => null);
    }

    s.AddOcelot();
});
```

简易方式意味着您只需设计提供者类，并为 Ocelot 核心中的默认 `ServiceDiscoveryProviderFactory` 指定 `ServiceDiscoveryFinderDelegate` 对象。

更复杂的设计意味着您设计了提供者类和提供者工厂类。之后，您需要将 `IServiceDiscoveryProviderFactory` 接口添加到 DI 容器中，删除默认注册的 `ServiceDiscoveryProviderFactory` 类。请注意，在这种情况下，Ocelot 管道将不会默认使用 `ServiceDiscoveryProviderFactory`。此外，您无需在 `ServiceDiscoveryProvider` 属性的 `GlobalConfiguration` 设置中指定 `Type`。但您可以保留此 `Type` 选项以便与两种设计兼容。

1. `AddOcelot` 方法将默认的 ASP.NET 服务添加到 DI 容器中。您可以在配置服务时调用其他扩展的 `AddOcelotUsingBuilder` 方法来开发自己的自定义构建器。有关更多说明，请参见依赖注入功能中的“AddOcelotUsingBuilder 方法”部分。

2. “Consul 配置密钥”功能是作为版本 7.0.0 的一部分在问题 346 中请求的。

3. “Consul 服务构建器”的自定义功能是作为修复 bug 954 的一部分实现的，并在版本 23.3 中发布。

## Service Fabric

如果您在 Azure Service Fabric 中部署了服务，通常会使用命名服务来访问这些服务。

以下示例展示了如何设置一个在 Service Fabric 中工作的 Route。最重要的是 `ServiceName`，它由 Service Fabric 应用程序名称和特定的服务名称组成。我们还需要在 `GlobalConfiguration` 中设置 `ServiceDiscoveryProvider`。以下示例显示了一个典型的配置，假设 Service Fabric 运行在本地主机上，命名服务运行在 19081 端口。

```json
{
  "Routes": [
    {
      "DownstreamScheme": "http",
      "DownstreamPathTemplate": "/api/values",
      "UpstreamPathTemplate": "/EquipmentInterfaces",
      "UpstreamHttpMethod": [ "Get" ],
      "ServiceName": "OcelotServiceApplication/OcelotApplicationService"
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://ocelot.com",
    "RequestIdKey": "OcRequestId",
    "ServiceDiscoveryProvider": {
      "Host": "localhost",
      "Port": 19081,
      "Type": "ServiceFabric"
    }
  }
}
```

如果您使用的是无状态/来宾 exe 服务，Ocelot 将能够通过命名服务进行代理，而无需其他操作。然而，如果您使用的是有状态/actor 服务，必须在客户端请求中发送 `PartitionKind` 和 `PartitionKey` 查询字符串值，例如：

```
GET http://ocelot.com/EquipmentInterfaces?PartitionKind=xxx&PartitionKey=xxx
```

Ocelot 无法为您处理这些值。

### 服务名称中的占位符 [1]

在 Ocelot 中，您可以在 `UpstreamPathTemplate` 和 `ServiceName` 中插入变量占位符，格式为 `{something}`。

**重要提示：** 占位符变量必须在 `DownstreamPathTemplate`（或 `ServiceName`）和 `UpstreamPathTemplate` 中都存在。具体来说，`UpstreamPathTemplate` 应包括来自 `DownstreamPathTemplate` 和 `ServiceName` 的所有占位符。如果没有这样做，将导致 Ocelot 因验证错误而无法启动，这些错误会被记录下来。

一旦通过验证阶段，Ocelot 将在处理的每个请求中用 `DownstreamPathTemplate` 和/或 `ServiceName` 中的值替换 `UpstreamPathTemplate` 中的占位符。因此，`ServiceName` 中的占位符行为类似于占位符功能，但在处理过程中考虑了 `ServiceName` 属性。

### 占位符示例

以下是 Service Fabric 服务名称中版本变量的示例。

假设您有如下的 `ocelot.json`：

```json
{
  "Routes": [
    {
      "UpstreamPathTemplate": "/api/{version}/{endpoint}",
      "DownstreamPathTemplate": "/{endpoint}",
      "ServiceName": "Service_{version}/Api"
    }
  ],
  "GlobalConfiguration": {
    "BaseUrl": "https://ocelot.com",
    "ServiceDiscoveryProvider": {
      "Host": "localhost",
      "Port": 19081,
      "Type": "ServiceFabric"
    }
  }
}
```

当您发出请求时：`GET https://ocelot.com/api/1.0/products`

那么 Service Fabric 的请求将是：`GET http://localhost:19081/Service_1.0/Api/products`

### 参考文献

1. “服务名称中的占位符 1”功能是在问题 721 中请求的，并通过 PR 722 在版本 13.0.0 中交付。

## 分布式追踪

本页面详细介绍了如何使用 Ocelot 执行分布式追踪。

#### OpenTracing

Ocelot 提供了 OpenTracing API for .NET 项目的追踪功能。Ocelot 集成的代码可以在 `Ocelot.Tracing.OpenTracing` 项目中找到。

下面的示例使用了 C# 客户端库 Jaeger 提供的 tracer。在 Ocelot 中添加 OpenTracing 服务时，我们必须调用 `AddOpenTracing()` 扩展方法，该方法由 `AddOcelot()` 返回的 `OcelotBuilder` 提供，如下所示：

```csharp
services.AddSingleton<ITracer>(sp =>
{
    var loggerFactory = sp.GetService<ILoggerFactory>();
    Configuration config = new Configuration(context.HostingEnvironment.ApplicationName, loggerFactory);

    var tracer = config.GetTracer();
    GlobalTracer.Register(tracer);
    return tracer;
});

services
    .AddOcelot()
    .AddOpenTracing();
```

然后，在 `ocelot.json` 中，为需要追踪的 Route 添加以下内容：

```json
"HttpHandlerOptions": {
  "UseTracing": true
}
```

现在，当调用该 Route 时，Ocelot 将会把追踪信息发送到 Jaeger。

**OpenTracing 状态**  
OpenTracing 项目在 2022 年 1 月 31 日被归档（参见文章）。Ocelot 团队将决定是否迁移到 OpenTelemetry，这也是高度期望的。

#### Butterfly

Ocelot 还提供了来自优秀 Butterfly 项目的追踪功能。Ocelot 集成的代码可以在 `Ocelot.Tracing.Butterfly` 项目中找到。

要使用 Butterfly 进行追踪，请阅读 Butterfly 文档。

在 Ocelot 中，如果要追踪 Route，需要添加 NuGet 包：

```bash
Install-Package Ocelot.Tracing.Butterfly
```

在 `ConfigureServices` 方法中添加 Butterfly 服务时，我们必须调用 `AddButterfly()` 扩展方法，该方法由 `AddOcelot()` 返回的 `OcelotBuilder` 提供，如下所示：

```csharp
services
    .AddOcelot()
    // This comes from Ocelot.Tracing.Butterfly package
    .AddButterfly(option =>
    {
        // This is the URL that the Butterfly collector server is running on...
        option.CollectorUrl = "http://localhost:9618";
        option.Service = "Ocelot";
    });
```

然后，在 `ocelot.json` 中，为需要追踪的 Route 添加以下内容：

```json
"HttpHandlerOptions": {
  "UseTracing": true
}
```

现在，当调用该 Route 时，Ocelot 将会把追踪信息发送到 Butterfly。

**参考文献**  
1. `AddOcelot` 方法将默认 ASP.NET 服务添加到 DI 容器。您可以在配置服务时调用其他扩展的 `AddOcelotUsingBuilder` 方法，以开发自己的自定义 Builder。有关更多说明，请参见“AddOcelotUsingBuilder 方法”部分。

## WebSockets 支持

#### WebSockets 标准
- **WHATWG**: WebSockets 标准
- **IETF**: WebSocket 协议

Ocelot 支持 WebSockets 的代理功能，这一功能是在 issue 212 中提出的。

要使 WebSocket 代理在 Ocelot 中工作，需要执行以下步骤：

1. 在 `Configure` 方法中启用 WebSockets：

```csharp
Configure(app =>
{
    app.UseWebSockets();
    app.UseOcelot().Wait();
});
```

2. 在 `ocelot.json` 中添加以下配置来使用 WebSockets 代理 Route：

```json
{
  "UpstreamPathTemplate": "/",
  "DownstreamPathTemplate": "/ws",
  "DownstreamScheme": "ws",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 5001 }
  ]
}
```

这个配置将使 Ocelot 匹配任何来自 `/` 的 WebSocket 流量，并将其代理到 `localhost:5001/ws`。具体来说，Ocelot 将接收来自上游客户端的消息，将其代理到下游服务，接着接收下游服务的消息，并将其代理回上游客户端。

**参考文献**  
- WHATWG: [WebSockets 标准](https://whatwg.org/specs/websockets/)
- Mozilla Developer Network: [WebSocket API](https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API)
- Microsoft Learn: [ASP.NET Core 中的 WebSockets 支持](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/websockets)
- Microsoft Learn: [在 .NET 中的 WebSockets 支持](https://learn.microsoft.com/en-us/dotnet/framework/network-programming/websockets)

#### SignalR 支持

Ocelot 支持代理 SignalR，这一功能是在 issue 344 中提出的。要使 SignalR 代理工作，需要执行以下步骤：

1. 安装 SignalR 客户端 NuGet 包：

```bash
Install-Package Microsoft.AspNetCore.SignalR.Client
```

虽然该包已弃用，但新版本仍然从源代码构建。因此，SignalR 是 ASP.NET 框架的一部分，可以像这样引用：

```xml
<ItemGroup>
  <FrameworkReference Include="Microsoft.AspNetCore.App" />
</ItemGroup>
```

有关框架兼容性的更多信息，请参见 [Use ASP.NET Core APIs in a class library](https://learn.microsoft.com/en-us/aspnet/core/host-and-deploy/aspnet-core-in-a-windows-service?view=aspnetcore-5.0).

2. 配置应用程序以使用 SignalR：

```csharp
public void ConfigureServices(IServiceCollection services)
{
    services.AddOcelot();
    services.AddSignalR();
}
```

注意 WebSockets 的传输级配置，确保配置允许 WebSockets 连接。

3. 在 `ocelot.json` 中添加以下配置来代理 SignalR Route。请注意，普通的 Ocelot 路由规则仍适用，关键是将 scheme 设置为 `ws`。

```json
{
  "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE", "OPTIONS" ],
  "UpstreamPathTemplate": "/gateway/{catchAll}",
  "DownstreamPathTemplate": "/{catchAll}",
  "DownstreamScheme": "ws",
  "DownstreamHostAndPorts": [
    { "Host": "localhost", "Port": 5001 }
  ]
}
```

#### 安全 WebSocket (wss)

如果您定义了一个使用安全 WebSocket 协议的路由，请使用 `wss` scheme：

```json
{
  "DownstreamScheme": "wss"
}
```

请注意，您可以对 SignalR 和 WebSockets 使用 WebSocket SSL。

**参考文献**  
- Microsoft Learn: [使用 TLS/SSL 保护连接](https://learn.microsoft.com/en-us/aspnet/core/security/https)
- IETF: [WebSocket URIs](https://tools.ietf.org/html/rfc6455)

#### SSL 错误

如果您希望忽略 SSL 警告（错误），请在 Route 配置中设置以下内容：

```json
{
  "DownstreamScheme": "wss",
  "DangerousAcceptAnyServerCertificateValidator": true
}
```

但我们不推荐这样做！请阅读配置文档中的官方说明，了解有关 SSL 错误的最佳实践。

**注意**  
- `wss` scheme 的假验证器由 PR 1377 添加，作为 issue 1375、1237 等的一部分。此功能用于自签名 SSL 证书，在 20.0 版本中可用。未来版本可能会移除或重做此功能。

#### 支持与不支持

**支持**  
- 负载均衡
- 路由
- 服务发现

这意味着您可以设置运行 WebSockets 的下游服务，并在 Route 配置中设置多个 `DownstreamHostAndPorts`，或者将您的 Route 连接到服务发现提供者，然后负载均衡请求。

**不支持**  
遗憾的是，很多 Ocelot 功能并非 WebSocket 专用，例如头部和 HTTP 客户端功能。以下是一些不适用的功能：

- 追踪
- 请求 ID
- 请求聚合
- 限速
- 服务质量
- 中间件注入
- 头部转换
- 委托处理程序
- 声明转换
- 缓存
- 身份验证
- 授权

我们不能 100% 确保该功能在实际使用中的表现，请务必进行充分测试！

#### 未来计划

WebSockets 和 SignalR 正在由 .NET 社区积极开发，因此需要定期关注官方文档中的趋势和版本更新：

- [WebSockets 文档](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/websockets)
- [SignalR 文档](https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction)

我们无法为开发提供建议，但欢迎在存储库的讨论区提问、获取代码示例或报告问题。Ocelot 团队考虑到当前的 WebSockets 实现已经过时（基于 WebSocketsProxyMiddleware 类），我们有强烈意图迁移或至少重新设计此功能，参见 issue 1707。

**参考文献**  
1. 如果有人请求，我们可能会处理基本身份验证相关的功能。