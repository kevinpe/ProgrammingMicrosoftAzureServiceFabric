
# 第二章： 无状态服务

在第一章我们已经实现了一个不处理任何用户请求的后台服务。虽然后台服务非常有用，但是大多数服务需要接受并处理用户的请求。本章我们将学习如何使用无状态服务处理用户请求。首先我们会学习如何添加ASP.NET 5 Web服务到我们的应用；接下来，我们将学习如何为我们的服务实现一些通用的通信栈来处理不同协议的客户端请求。

## 实现ASP.NET 5应用

基于云的应用通常都会有Web前端，所以Service Fabric包含了ASP.net Web API模板。通过该模板，我们可以很容易的添加Web API到Service Fabric应用中。在这一节，我们会创建一个带ASP.Net 5 Web API的应用。然后我们会部署它到本地集群和Azure集群上。

创建ASP.NET 5 Web API非常的简单，只需要在添加服务的时候选择ASP.NET 5 Web API模板，一个可部署的Web API的站点就被创建好了。通过下面步骤可以添加ASP.NET 5 Web API到我们的Service Fabric应用中。

1. 创建一个新的Service Fabric应用。

2. 在创建服务的窗口中，选择ASP.Net 5 Web API模板并点击“OK”继续，如图2-1.
    ![](/images/chapter2/Image_1.jpg)

3. 当应用被创建，发布应用到我们本地集群。 默认情况下，服务会是一个单分区单实例的服务，并监听在一个自动分配的端口上。打开Service Fabrc Explorer，从应用视图里面我们可以查看应用应用实例所在的位置和监听的端口。如图2-2，服务实例是被运行在节点2，监听地址是Http://+:34001 。
    ![](/images/chapter2/Image_2.jpg)

4. 打开浏览器并访问http://localhost:34001 (在不同系统上，端口可能不同)。 我们可以看见一个请求是被发送到web API的values控制器上，如图2-3。
    ![](/images/chapter2/Image_3.jpg)

5. 现在修改我们Service FAbric应用项目下的ApplicationManifest.xml 文件。修改StatelessService元素的InstanceCount属性为-1. 我们应用实例会被部署到每一个节点上。
    ```
    <Service Name="Web1Service">
        <StatelessService ServiceTypeName="Web1Type" InstanceCount="-1">
          <SingletonPartition />
        </StatelessService>
    </Service>
    ```

6. 保存修改并重新部署我们的应用。

    当应用被部署后，我们可以看到每一个节点上运行了一个应用实例，如图2-4. 每一个实例监听在一个自动分配的独特的端口上。
    ![](/images/chapter2/Image_4.jpg)

## 无状态服务的可扩展性和可用性

服务的多个实例监听在不同的端口上引入了一个新的问题。 我们可以使用每个副本的特定的地址连接每一个实例，我们可以用一个客户端程序查询特定实例的连接地址。然后，当用户使用web浏览器连接这些服务的时候，他们没有这实例的地址信息，也不具有获取这些实例的地址的能力。理想情况是，他们应该总是连接某个预定义的端口（如80），然后被重定向到工作实例上。对于这种情况，负载均衡是解决可扩展性和可用性的一种可行方案。

### 可用性
由于浏览器不知道如何使用命名服务来查找和连接健康的实例，因此这部分逻辑必须在服务端实现。 在服务端可以通过一个包含实例发现逻辑的网关，或者是一个能够路由用户请求到健康节点的负载均衡器来完成这项功能。这里我们将重点放在负载均衡器上，因为Azure提供了这个功能。 通常，负载均衡器探测实例的状态，并分配流量到健康节点上，如图2-5。
![](/images/chapter2/Image_5.jpg)

在第五章“服务部署和升级”，我们将更多关于如何在我们的Azure服务实例前配置负载均衡器。

### 可扩展性
由于负载均衡器可以自动的在健康实例之间分配流量， 它同时也是的服务变得可扩展。

在某些场景下，我们可以对无状态服务进行分区，已达到隔离客户的目的。例如，当我们想要发送不同类型的工作负载到不同类型节点时。但是，分区更主要是为了隔离客户，而非提供可扩展性。第五章将会对无状态服务的分区做更多讨论。
扩展的另外一个层面是部署多个应用实例。 例如，我们可以为每个客户部署一个应用实例，基于特定客户的工作负载来扩展服务实例。这种扩展机制在第五章我们会有更多的讨论。


## 通信栈的实现
一个托管ASP.NET 5 Web服务并不是唯一响应用户请求的方式。 正如第一章提到的，Service Fabric允许我们为不同的通信协议实现不同的定制通信栈。通过实现ICommunicationListener接口来定义通信机制。
接下来，我们将实现一个简单的提供加减操作的计算服务，我们会为这个服务实现三种不同的通信栈。

### 默认通信栈
Service Fabric已经提供了一个基于RPC代理的通信栈，第一个版本的计算服务将使用这个缺省的通信栈。

#### 第一版
1. 运行Visual Studio 2015.。创建一个新的Service Fabric应用CalculatorApplication，应用包含一个无状态服务CalculatorService。

2. 在CalculatorService项目中，定义一个叫ICalculatorService服务合约 。 注意：服务接口需要继承自Microsoft.ServiceFabric.Services.Remoting名字空间下的IService接口。
    ```
    public interface ICalculatorService: IService
    {
        Task<int> Add(int a, int b);
        Task<int> Subtract(int a, int b);
    }
    ```

3. 修改CalculatorService 实现服务合约，如下面代码段所示。 在下面代码段中有许多值得注意的地方，首先， 如果我们服务不需要一个长期运行的后端逻辑，我们可以移除RunAsync方法；第二，为了使用缺省的远程通信栈，我们需要创建一个新的ServiceRemotingListener 实例。
    ```
    internal sealed class CalculatorService : StatelessService, ICalculatorService
    {
        public Task<int> Add(int a, int b)
        {
            return Task.FromResult<int>(a + b);
        }
        public Task<int> Subtract(int a, int b)
        {
            return Task.FromResult<int>(a - b);
        }
        protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
        {
            return new[]
            {
                new ServiceInstanceListener(initParams =>
                    new ServiceRemotingListener<ICalculatorService>(initParams, this))
            };
        }
    }
    ```

    >和Azure Cloud Service的RunAsync方法对比
    >
    >如果熟悉 Azure Cloud Service，你可能还记得Azure Cloud Service要求RunAsyn方法始终保持运行。如果>RunAsync方法失败或者是退出，该实例就会被认为失效，Cloud Service 将尝试重新启动该实例。 对Service Fabric没有这要的要求，在Service Fabric的RunAsync可以完成任务退出，或者是完全忽略RunAsync的实现。而Service Fabric实例可以继续监听，接受和处理客户端请求。

4. 重新编译我们的所有项目，右键点击CalculatorApplication项目并选择发布。 在发布对话框中，选择本地配置， 如图2-6。接下来，点击发布按钮发布应用到本地集群。
    ![](/images/chapter2/Image_6.jpg)

5. 添加一个新的控制台应用CalculatorClient  到我们的解决方案里。

6. 右键点击CalculatorClient 项目，选择“Manage NuGet Packages ”菜单。在NuGet包管理器中，选择包含Prelease check box“”并搜索Microsoft.ServiceFabric.Services， 然后点击安装。如图2-7.
    ![](/images/chapter2/Image_7.jpg)

7. 添加一个链接指向CalculatorService项目下的ICalculatorService.cs文件，把服务合约的定义引入客户端项目。 一个比较好的实现是在一个共享的assebly中定义合约。 本例中，我了简单我们直接引用了该文件。

    > __注意： 添加文件链接__
    >
    >若要添加链接到另一源文件，请右键单击项目并选择“添加\现有项”菜单项。然后，浏览到文件并选择它。或者是直接单击“添加”按钮，单击“添加”按钮旁边的“小三角形”下拉菜单，然后选择“添加”作为链接添加到原始文件的链接（而不是复制）。所有原始文件的更新都反映在链接文件中。

8. 右键点击CalculatorClient项目，在菜单中选择属性，切换到编译标签，修改目标平台成x64。
9. 打开Program.cs文件，导入下列名字空间：
    ```
    using CalculatorService;
    using Microsoft.ServiceFabric.Services.Remoting.Client;
    ```
10. 如下代码段所示修改Main方法。
    ```
    static void Main(string[] args)
    {
      var calculatorClient = ServiceProxy.Create<ICalculatorService>
      (new Uri("fabric:/CalculatorApplication/CalculatorService"));
        var result = calculatorClient.Add(1, 2).Result;
        Console.WriteLine(result);
        Console.ReadKey();
    }
    ```

11. 客户端程序已经准备好了，我们可以试一试了。 右键点击CalculatorClient项目，并选择Debug\Start菜单。 我们可以看见正确的结果输出，这里输出的是3。有可能会有一些可以忽略的安全警告消息。

这不是特别令人兴奋，但还是有一些有趣的地方。当客户端连接到服务，它使用URI地址是“fabric:/CalculatorApplication/CalculatorService” ，不是我们熟悉的HTTP或TCP服务地址。这个地址被命名服务转换为服务侦听器端点。图2-8显示本地集群的服务情况（表格一些列可编辑删除）。图中显示，有几个系统服务在运行，包括clustermanagerservice，failovermanagerservice，和namingservice。该图还显示，我们单服务分区下运行五个副本 ，每一个监听在一个独特的net.tcp 地址上。
![](/images/chapter2/Image_8.jpg)

#### Service Fabric应用的模型

在我们队计算服务做更多修改之前，为了更多的理解service fabric应用的结构，让我们检查一下service fabric应用的模型。
我们已经知道Service Fabric应用是由服务组成， Service Fabric服务由代码，配置和数据组成。
* __代码__ 服务的可执行二进制文件，这些二进制文件被打包到代码包中。
* __配置__ 运行时被加载的设置。 这些设置在服务项目的Settings.xml文件中被定义，他们被打包进配置包。
* __数据__  数据保护了任何被服务使用的静态数据，数据会被打包到数据包中。

图2-9描述了Service Fabric应用模型
![](/images/chapter2/Image_9.jpg)

Service Fabric应用模型是被一个应用程序清单和许多服务清单清楚描述。应用清单描述了应用的结构，服务清单描述了服务中的代码，配置，和数据。应用程序清单和服务清单具有相关的版本号，分别用于标识应用程序类型和服务类型的版本。

##### 服务清单

我们可以在项目的packageroot文件夹下找到服务的清单。清单是一个具有以下关键元素的xml文件：

* __ServiceTypes__ 定义了在服务代码包中支持的服务类型。例如，我们的计算服务清单中声明了它支持一个CalculatorServiceType服务类型。
    ```
    <ServiceTypes>
            <StatelessServiceType ServiceTypeName="CalculatorServiceType" />
    </ServiceTypes>
    ```

* __CodePackage__  描述了代码包中包含的二进制可执行文件。 一个服务清单可以包含多个代码包。当一个服务类型被Service Fabric实例化的时候， 在清单中的所有代码都被激活，他们的入口函数将被调用。 调用这些入口点来创建一个新的服务进程，这些进程将注册它支持的服务类类型并启动。例如，查看我们的计算器服务项目的CodePackage 元素：
    ```
      <CodePackage Name="Code" Version="1.0.0.0">
          <EntryPoint>
                    <ExeHost>
                             <Program>CalculatorService.exe</Program>
                    </ExeHost>
           </EntryPoint>
      </CodePackage>
    ```

  入口点被定义为CalculatorService.exe 的可执行文件。 
  在我们的例子中，入口点定义为一个可执行的叫CalculatorService.exe。我们的计算服务代码不过是一个引用了Service Fabric库的控制台应用程序。从项目的Main方法中，我们可以看到程序向Service Fabric注册支持的服务类型：
    ```
    using (FabricRuntime fabricRuntime = FabricRuntime.Create())
    {
      fabricRuntime.RegisterServiceType("CalculatorServiceType", typeof(CalculatorService));
      ...
    }
    ```

* __ConfigPackage__ 服务清单可包含许多ConfigPackage元素，每一个都是由名称属性标识。服务项目的PackageRoot文件夹下，每一个ConfigPackage元素应该有一个具有相同的名称的文件夹。在每个文件夹下有一个settings.xml文件，其中包含可以在运行时加载键/值对的设置。

* __DataPackage__  服务清单可包含多个DataPackage元素。每个DataPackage元素在PackageRoot有一个对应的文件夹。每个文件夹包含一些可以由该服务使用的任意静态数据文件。

##### 应用程序清单
应用程序清单描述应用程序的整体结构。具体来说，它描述了在应用程序中有什么样的服务，以及如何将这些服务应该布置到Service Fabric集群。在本书后面，您将了解本文档中的高级元素和属性。现在，只看其中的几个：

* __ServiceManifestImport__ 这个元素定义对组成应用的服务清单的引用
* __DefaultServices__  默认服务会在应用实例化的时候自动被创建。除了自动实例化外，默认服务和其它服务并没有任何区别。


#### 第二版
为了能够知道那个服务实例处理了这个请求，现在我们修改服务返回副本ID和计算结果。 我们首先需要修改服务合约返回字符串，而不是整数。

1. 修改ICalculatorService 接口的返回类型为Task<string>。
    ```
    public interface ICalculatorService: Iservice
    {
        Task<string> Add(int a, int b);
        Task<string> Subtract(int a, int b);
    }
    ```
2. 在CalculatorService 类中修改计算方法的实现。下面的代码段演示了如何使用StatelessService 的ServiceInitializationParameters 属性访问当前实例（副本）ID。
    ````
    public Task<string> Add(int a, int b)
    {
        return Task.FromResult<string>(string.Format("Instance {0} returns: {1}",
            this.ServiceInitializationParameters.InstanceId, a + b));
    }
    public Task<string> Subtract(int a, int b)
    {
        return Task.FromResult<string>(string.Format("Instance {0} returns: {1}",
            this.ServiceInitializationParameters.InstanceId,
            a - b));
    }
    ```
3. 因为我们需要修改服务合约， 我们有理由把修改后的服务称为版本2。我们需要修改服务清单和应用清单。修改CalculatorService项目PackageRoot目录下的ServiceManifest.xml 文件，修改ServiceManifest和CodePackage元素的Version属性为2.0.0.0。 代码包和服务的版本都会被更新为2.0.0.0.

4. 编辑CalculatorApplication项目的ApplicationManifest.xml，修改ApplicationManifest  元素的ApplicationTypeVersion 属性和ServiceManifestRef  元素的ServiceManifestVersion 属性为“2.0.0.0”. 这个将更新应用版本为“2.0.0.0”，并引用新版本的服务。

5. 重新编译部署应用到我们的本地节点。

6. 修改客户端，循环调用服务。
    ```  
    while (true)
    {
          var calculatorClient = ServiceProxy.Create<ICalculatorService>(new
                  Uri("fabric:/CalculatorApplication/CalculatorService"));
          var result = calculatorClient.Add(1, 2).Result;
          Console.WriteLine(result);
          Thread.Sleep(3000);
    }
    ```  
7. 运行客户端。客户端将随机的联系实例，并接受返回结果。 如图2-10，客户端连接到实例130897044575029205。
    ![](/images/chapter2/Image_10.jpg)
    
    在Service Fabric浏览器中，打开服务分区查看所有副本。根据客户端输出的实例ID可以找到托管的节点。 在右边面板中选择该节点，点击Actions按钮的“Deactivate”菜单。 模拟节点崩溃。 如图2-11，节点3（托管了副本130897044575029205 ）被选择并关闭，（在不同的环境中有不同的副本ID ）
    ![](/images/chapter2/Image_11.jpg)

8. 一旦节点被关闭，客户端与原来的服务实例的连接被断开，并自动重连到另一个健康的实例。图2-12显示了本例中中的客户端从新连接到了ID为130897044575029205的副本。这里实际上就是故障转移机制。尽管原来连接的服务实例已经下线，当客户端没有感受到被中断，并自动的从新连接到一个健康的实例。
    ![](/images/chapter2/Image_12.jpg)


> 重置集群
>
> 关闭节点有时导致本地群变得不稳定。如果你遇见然后意料之外的错误，请通过重置集群恢复集群到健康状态再重试。

### WCF 通信栈
接下来，我们将查看Service Fabric实现的另外一种通信栈： WCF通信栈。 在这个练习里，我们将修改计算服务使用WCF为客户端提供服务。

> __注意： WCF故障排除__
>
>这个练习不是为了教WCF，所以在这里许多WCF相关的细节不会被提供。如果你对WCF比较熟悉，下面这些代码没有什么惊奇之处。 >如果你遇到一些问题和WCF有点生疏，记住WCF的基本概念： 地址，绑定，和合约。 大部分的WCF问题是由以上三部分的不匹配引>起的。 你可以从https://msdn.microsoft.com/library/ms731082(v=vs.110).aspx. 找到更多关于WCF的资料

#### 第三版

1. 修改CalculatorService类， 用WCF通信栈替换掉默认的通信栈。EndpointResourceName指向了服务清单文件中相应的端点配置。 私有方法CreateListenBinding创建一个端点的绑定，我们接下来将实现它。
    ```
    protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
    {
        return new[]
        {
          new ServiceInstanceListener(initParams =>
              new WcfCommunicationListener(initParams, typeof (ICalculatorService), this)
            {
                EndpointResourceName = "ServiceEndpoint",
                Binding = this.CreateListenBinding()
            })
        };
    }
    ```

2. CreateListenBinding方法。下面的实现中我们直接硬编码值。在实际的产品实现中，我们可以把让这些参数从服务的配置中读取。
    ```
    private NetTcpBinding CreateListenBinding()
    {
      NetTcpBinding binding = new NetTcpBinding(SecurityMode.None)
              {
                     SendTimeout = TimeSpan.MaxValue,
                     ReceiveTimeout = TimeSpan.MaxValue,
                     OpenTimeout = TimeSpan.FromSeconds(5),
                     CloseTimeout = TimeSpan.FromSeconds(5),
                     MaxConnections = int.MaxValue,
                     MaxReceivedMessageSize = 1024 * 1024
                };
            binding.MaxBufferSize = (int)binding.MaxReceivedMessageSize;
            binding.MaxBufferPoolSize = Environment.ProcessorCount * binding.MaxReceivedMessageSize;

            return binding;
    }
    ```

3. 从ICalculatorService 移除IService基类接口。

4. 更新服务和应用清单的版本号为3.0.0, 重新编译部署应用。

5. 添加一个新的Client类到CalculateClient项目。
    ```
    public class Client : ServicePartitionClient<WcfCommunicationClient<ICalculatorService>>,ICalculatorService
    {
        public Client(WcfCommunicationClientFactory<ICalculatorService> clientFactory, Uri serviceName)
            : base(clientFactory, serviceName)
        {
        }
        public Task<string> Add(int a, int b)
        {
            return this.InvokeWithRetryAsync(client => client.Channel.Add(a, b));
        }
        public Task<string> Subtract(int a, int b)
        {
            return this.InvokeWithRetryAsync(client => client.Channel.Subtract(a, b));
        }
    }
    ```

6. 修改客户端程序的Main方法:
    ```
    Uri ServiceName = new Uri("fabric:/CalculatorApplication/CalculatorService");
    ServicePartitionResolver serviceResolver = new ServicePartitionResolver(() => new FabricCliet());

    NetTcpBinding binding = CreateClientConnectionBinding();
    Client calcClient = new Client(new WcfCommunicationClientFactory<ICalculatorService>
           (serviceResolver, binding, null), ServiceName);
    Console.WriteLine(calcClient.Add(3, 5).Result);
    Console.ReadKey();
    ```

7. 实现CreateClientConnectionBinding方法，并为客户端创建匹配的绑定。
    ```
    private static NetTcpBinding CreateClientConnectionBinding()
    {
        NetTcpBinding binding = new NetTcpBinding(SecurityMode.None)
          {
              SendTimeout = TimeSpan.MaxValue,
              ReceiveTimeout = TimeSpan.MaxValue,
              OpenTimeout = TimeSpan.FromSeconds(5),
              CloseTimeout = TimeSpan.FromSeconds(5),
              MaxReceivedMessageSize = 1024 * 1024
          };

        binding.MaxBufferSize = (int)binding.MaxReceivedMessageSize;
        binding.MaxBufferPoolSize = Environment.ProcessorCount * binding.MaxReceivedMessageSize;
        
      return binding;
    }
    ```
8. 编译和运行客户端，我们可以看到结果成功的从服务返回。


### 定制通信栈
在本节的最后一个练习里，我们将使用OWIN托管ASP.NET Web API的通信栈。如果对这些术语不是很熟悉，下面将会对这些概念做一个简单的介绍。 一旦你准备好，我们将实行一个带有Web API的计算服务。

#### OWIN, ASP.NET Web API, 和项目Katana

通常，ASP.NET是和IIS紧密的绑在一起的。 OWIN (OWIN 2015),  Open Web Interface for .NET,的提出是为了解耦ASP.NET应用和web服务器。这个使得我们可以托管ASP.NET应用到任何遵循OWIN规范的web服务器，或者是我们的进程自己作为Web服务器。 图2-13，显示了OWIN的高层架构：　宿主是托管进程，服务器接受客户端的请求并响应。　在请求到达基于各种框架之上的应用之前，请求在可定制的中间件栈之间被传递。
![](/images/chapter2/Image_13.jpg)

Katana项目是微软实现的OWIN组件。 在本例中，我们将使用Katana’s OWIN 自托管的实现作为托管和OWIN HttpListener(基于the .NET Framework HttpListener)作为服务器。

#### OWIN自托管的ICommunicationListener
如前面所述，为了实现通信栈，我们只需要提供一个ICommunicationListener接口的实现。
```
public interface ICommunicationListener
{
    void Abort();
    Task CloseAsync(CancellationToken cancellationToken);
    Task<string> OpenAsync(CancellationToken cancellationToken);
}
```

我们应该在OpenAsync 方法中初始化并开始栈，在Abort 和CloseAsync 方法中关闭栈。

OpenAsync设置服务器需要监听的地址，并开始接受处理请求。
CloseAsync停止接受新的请求，结束在处理的请求，并结束。
Abort 立即取消和停止。


ICommunicationListener如何与OWIN self-host and ASP.NET Web API协同工作？ 在这个例子里，通信监听者将运行OWIN服务器，OWIN服务器将通过IAppBuilder接口初始化ASP.NET Web API。如果2-14.
![](/images/chapter2/Image_14.jpg)

>__注意：本例来源__
>
>下面的例子是受Vaclav Turecek 所启发 azure.microsoft.com: https://azure.microsoft.com/documentation/articles/service-fabric-reliable-services-communication-webapi/.

#### 第四版

1. 创建一个带WebCalculatorService 无状态服务的WebCalculatorApplication service Fabric应用。
2. 在WebCalculatorService项目中，添加对Microsoft.AspNet.WebApi.OwinSelfHost NuGet包的引用。

3. 下一步，我们建建立ASP.NET API。 这里没有什么特殊的地方。 我们只需要建立一个普通的ASP.NET API结构的类，像其它ASP.NET API项目一样。 第一，在WebCalculatorService 项目下创建下列目录。
  * App_Start
  * Controllers

4. 在App_Start 目录下的FormatterConfig.cs 中添加一个基本Web API配置类
    ```
    namespace WebCalculatorService
    {
        using System.Net.Http.Formatting;
        public static class FormatterConfig
        {
            public static void ConfigureFormatters(MediaTypeFormatterCollection formatters)
            {
            }
        }
    }
    ```

5. 在App_Start 目录下的RouteConfig.cs中添加一个基本的路由配置类
    ```
    namespace WebCalculatorService
    {
        using System.Web.Http;

        public static class RouteConfig
        {
            public static void RegisterRoutes(HttpRouteCollection routes)
            {
                routes.MapHttpRoute(
                      name: "CalculatorApi",
                      routeTemplate: "api/{action}",
                      defaults: new { controller = "Default" }
                  );
            }
        }
    }
    ```

6. 在控制器目录下，添加一个DefaultController控制器，它提供计算器的方法。
    ```
    namespace WebCalculatorService.Controllers
    {
        using System.Collections.Generic;
        using System.Web.Http;

        public class DefaultController : ApiController
        {
            [HttpGet]
            public int Add(int a, int b)
            {
                return a + b;
            }

            [HttpGet]
            public int Subtract(int a, int b)
            {
                return a - b;
            }
        }
    }
    ```
7. 定义IOwinAppBuilder接口。
    ```
    namespace WebCalculatorService
    {
        using Owin;

        public interface IowinAppBuilder
        {
            void Configuration(IAppBuilder appBuilder);
        }
    }
    ```
8. 最后，定义一个Startup类，这个类将注册路由和一些其它配置。在这里是ASP.NET Web API框架被加入。
    ```
    namespace WebCalculatorService
    {
        using Owin;
        using System.Web.Http;
        
      public class Startup : IowinAppBuilder
        {
            public void Configuration(IAppBuilder appBuilder)
            {
                HttpConfiguration config = new HttpConfiguration();

                FormatterConfig.ConfigureFormatters(config.Formatters);
                RouteConfig.RegisterRoutes(config.Routes);

                appBuilder.UseWebApi(config);
            }
        }
    }
    ```

9. 在实现通信栈以前，我们先修改服务清单里的的服务端口配置。
    ```
    <Resources>
        <Endpoints>
            <Endpoint Name="ServiceEndpoint" Protocol="http" Port="80" Type="Input"/>
        </Endpoints>
    </Resources>
    ```
10. 当服务初始化时，Service Fabric会分配请求端口并为服务地址设置适当的ACL。如果没有标明端口，Service Fabric会自动从应用保留端口的范围里选择一个端口。当端口被指定是，SErvice Fabric将尝试分配指定端口。有时，如果端口冲突，分配就会失败。 因此，如果我们的服务不需要监听在固定端口，我们需要避免使用特定端口。

12. 然而，如果我们需要监听在某些特定端口（如80端口），我们需要留意在这个集群上部署的其他应用可能已经占用了这个端口。 为了避免这种冲突，Service Fabric会保证在单个节点上只会部署一个无状态服务。所以，在无状态服务里使用固定端口好像相对比较安全的。但是这里任然有一个问题：当我们使用本地集群，运行多个使用固定端口的无状态服务的副本，仍然会有端口冲突。避免这个冲突的方法是，在本地只允许单个实例，只有部署到云上时才进行扩展。 后面章节中，我们将看到如何为不同的环境设置不同的配置。

12. 添加我们的ICommunicationListener实现，添加新的OwinCommunicationListener类到服务项目。
    ```
    namespace WebCalculatorService
    {
        using Microsoft.ServiceFabric.Services;
        using System;
        using System.Threading.Tasks;
        using System.Fabric;
        using System.Threading;

        public class OwinCommunicationListener : IcommunicationListener
        {
            public void Abort()
            {
            }
            public Task CloseAsync(CancellationToken cancellationToken)
            {
            }
            public Task<string> OpenAsync(CancellationToken cancellationToken)
            {
            }
        }
    }
    ```

13. 定义一些局部变量和一个新的构造函数。
    ```
    private readonly IOwinAppBuilder startup;
    private IDisposable serverHandle;
    private string listeningAddress;

    public OwinCommunicationListener(IOwinAppBuilder startup)
    {
        this.startup = startup;
    }
    ```

14. 现在，让我们来实现OpenSync方法。首先，我们需要从ServiceInitializationParameters中读取端点设置，并基于端口构造HTTP URL. OpenSync方法将开始web服务器和返回服务器监听的地址。 Swrvice Fabric 注册返回的地址到命名服务，以便客户端可以查询到正确的通信地址。
    ```
    public Task<string> OpenAsync(CancellationToken cancellationToken)
    {
      EndpointResourceDescription serviceEndpoint = serviceInitializationParameters
                  .CodePackageActivationContext.GetEndpoint("ServiceEndpoint");
        int port = serviceEndpoint.Port;
        this.listeningAddress = String.Format("http://+:{0}/", port);

        this.serverHandle = WebApp.Start(this.listeningAddress, 
            appBuilder => this.startup.Configuration(appBuilder));
        string resultAddress = this.listeningAddress.Replace("+",
                  FabricRuntime.GetNodeContext().IPAddressOrFQDN);
        ServiceEventSource.Current.Message("Listening on {0}", resultAddress);
        return Task.FromResult(resultAddress);
    }
    ```

15. 最后，实现CloseAsync和Abort方法。
    ```
    public Task CloseAsync(CancellationToken cancellationToken)
    {
        this.StopWebServer();
        return Task.FromResult(true);
    }

    public void Abort()
    {
        this.StopWebServer();
    }

    private void StopWebServer()
    {
        if (this.serverHandle != null)
        {
            try
            {
                this.serverHandle.Dispose();
            }
            catch (ObjectDisposedException)
            {
                // no-op
            }
        }
    }
    ```
16. 我们定制的通信栈已经准备好了，只剩下在WebCalculatorService类中返回一个新的实例。
    ```  
    protected override IEnumerable<ServiceInstanceListener> CreateServiceInstanceListeners()
    {
      return new[]
      {
        new ServiceInstanceListener(initParams => new OwinCommunicationListener("webapp",
          new Startup(), initParams))
      };
    }
    ```

在部署新应用之前，简要的看看应用清单里的参数。 Service Fabric应用清单支持在部署是提供不同的配置参数。默认，Service Fabric Visual Studio 为每一个包含的服务生成一个实例数目的配置。在本例中， WebCalculatorService_InstanceCount参数控制我们的计算器服务的实例数目。在下面的代码段中，我们可以看到这个参数是如何被定义和应用的。
```
<ApplicationManifest ...>
    <Parameters>
        <Parameter Name="WebCalculatorService_InstanceCount" DefaultValue="1" />
    </Parameters>
    ...
    <DefaultServices>
        <Service Name="WebCalculatorService">
            <StatelessService ServiceTypeName="WebCalculatorServiceType"
                InstanceCount="[WebCalculatorService_InstanceCount]">
                <SingletonPartition />
            </StatelessService>
        </Service>
    </DefaultServices>
</ApplicationManifest>
```

当从Visual Studio发布Service Fabric应用时，我们可以选择一个配置（在PublishProfiles 目录下），每一个配置有一个不同的参数文件（在ApplicationParameters 目录下）。 如果没有参数很久被提供，或从参数文件里没有匹配的参数被找到，应用里面的缺省值会被使用。

对于我们的服务，我们可以设置Cloud.xml中的参数成-1. 这表明在每一个节点上部署一个实例，设置Local.xml文件中的参数为1，这表明在本地测试的时候创建一个实例。

接下来，在我们发布我们应用的时候，我们可以选择想用的配置和参数文件，如图2-15.
![](/images/chapter2/Image_15.jpg)
一旦应用部署完成，我们可以通过浏览器测试我们的方法。

http://localhost:80/webapp/api/add?a=1&b=2

http://localhost:80/webapp/api/subtract?a=8&b=3


## 附加信息
除了Service Fabric之外，本书中我们使用了多种技术。访问 https://msdn.microsoft.com/library/ms731082(v=vs.110).aspx 了解更多关于WCF的信息，访问 http://www.asp.net/ 了解更多关于ASP.NET,访问 http://owin.org/ 了解更多关于OWIN的信息。

2016年1月，微软宣布将使用基于新的.NET Core 1.0 的ASP.NET Core 1.0替换ASP.NET 5。 ASP.NET Core 1.0是一个全新的基于.NET core 的Web应用栈。 原来的ASP.NET将继续保持使用ASP.NET 4.6。写本书时，转换还在进行中。
