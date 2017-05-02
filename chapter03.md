# 第三章. 有状态服务

如第一章所述，许多服务本身就有状态，无论多高效和可靠，状态的管理都是挑战。 本章会首先指出在分布式系统中状态管理的困难点，然后再阐述Service Fabric是怎样解决这些问题的。 理解了其中的原理之后，会列出一系列实例来指导读者一步步创建有状态服务。

## Service Fabric状态管理

在分布式环境尤其是大型的分布式环境中，任何的消息都有可能丢失，任何节点都可能发生故障，任何的通信渠道都有可能堵塞。CAP定理（CAP theorem）表明，一致性、可用性和分区容错性不可能同时达成。然而，这并不意味着一个系统无法实现令人满意的一致性和可用性。Service Fabric的状态管理器使用了一系列策略来达到一致性和可用性的平衡。

## 有状态服务的架构

有状态服务使用由Service Fabric提供的可靠数据结构来保存状态。这些数据结构由Service Fabric来保存和复制。 有状态的Service Fabric 服务的架构如图3-1所示。 与无状态服务类似，有状态服务可以通过接口*ICommunicationListener*来接受客户端的请求。 当需要保存状态时，Reliable State Manager会将状态保存在可靠集合中，如reliable dictionaries和reliable queues。 为了保持可靠性和可用性，这些状态会由Transactional Replicator复制到副本中。

![](/images/chapter3/Image.jpg)

### 可靠集合

截止于本书撰写，Service Fabric提供了两种可靠集合：reliable dictionaries和reliable queues。 任意一种可靠集合都是状态提供程序。 这些数据结构暴露的接口使用与本地数据结构无异。 例如，*IReliableDictionary*的接口的使用与本地*Dictionary*无异。 Service Fabric使用了可插入式的基础架构使得更多的状态提供程序在将来得以实现。

### 可靠状态管理

Reliable State Manager管理着状态提供程序的生命周期。 尽管可靠集合可以像本地数据结构一样使用，但是它并不能直接实例化。 可靠集合中的数据结构需要被复制到多个节点上，因而这些数据结构实例需要协调一致地创建从而达到多节点上都可用。

无论继承自*StatefulService*还是*StatefulServiceBase*, 所有的有状态服务都提供了*IReliableStateManager*的接口用于管理状态。 可靠集合以字符串命名，你通过状态管理服务用命名来获取可靠集合的引用。 例如， 使用下列程序来获取名为*myDictionary*的reliable dictionary的引用：

	   var myDictionary = await this.StateManager.GetOrAddAsync<IReliableDictionary<string, long>>("myDictionary");  

Reliable State Manager还提供了事务性支持。 事务中所有操作都是原子操作，这就意味着无论操作的数目和更改的数据结构多少，所有更改要么同时生效要么同时取消。以下代码片段展示了在一个事务中如何在reliable dictionary中增加一对键值：

	using (var tx = this.StateManager.CreateTransaction())
	{
	    var result = await myDictionary.TryGetValueAsync(tx, "Counter-1");
	    await myDictionary.AddOrUpdateAsync(tx, "Counter-1", 0, (k, v) => ++v);
	    await tx.CommitAsync();
	}

### 事务复制器

事务复制器（Transactional Replicator）用于将可靠数据集复制到副本中。它也会调用日志系统来将一些状态保存在本地磁盘。

### 日志

所有的服务状态改变都会被保存到本地磁盘中的一个仅有附加权限的日志文件中。这使得状态信息在程序或者节点崩溃的情况下得以保存。通过回放这些日志，Service Fabric就可以恢复完整的服务状态。默认情况下，日志有两级：共享日志和专属日志。共享日志保存于节点级的工作目录下，适用于保存事务数据。专属日志是副本级日志，保存于服务的工作目录。状态信息最初是保存为共享日志，然后再慢慢地在后台转换为专属日志。

你可以通过配置日志服务来在不同的环境下提升性能和吞吐量。例如，在使用了固态硬盘(SSDs)的情况下，因为不存在寻道时间，你可以配置日志服务跳过保存为共享日志，直接将日志写入专属日志。

Reliable State Manager也会周期性地为整个服务状态做快照。这样能够精简状态更改日志从而节省磁盘空间。而在恢复过程中，可靠集合会从上一个已知的存档点恢复数据，然后Reliable State Manager再回放存档点后的状态更改日志，从而将状态完全恢复。

### 一致性

Service Fabric状态管理程序提供了即开即用的强一致性保障。事务的状态更改只有在一定数目的主副本全部成功应用之后，才视为生效。

所有数据通过主副本写入，而Service Fabric保证任何时间下一个partition只有一个主副本。数据改变将被复制到副本中，而当主副本失效后，从副本就会当选为新的主副本。数据可以从主副本或者从副本读出。

如果程序在某些场景下不需要强一致性，那么它在事务数据更新生效前就可以响应客户端需求。

## 一个简单的商店程序

现在我们可以开始写有状态服务的程序了！本章节剩余部分，我们将写一个简单的商店程序，用户可以再商店中购买商品。首先，我们使用Service Fabric有状态服务实现购物车服务。然后，我们将体验一下不同的分区(partition)方案。

然而在开始前，我需要声明一下一个分区一个购物车的方案不是一个好的选择，尤其是当一些分区的使用量不高的情况下。我将在第12章介绍一个更好的设计方案，同时为了简化本章的例子将继续使用此方案。

### 购物车服务

首先，我们要写一个购物车服务。在下述练习中，需要使用的WCF communication stack和WCF client.

1. 创建一个名为SimpleStoreApplication的Service Fabric应用，其中有状态服务(stateful service)名为ShoppingCartService。

2. 在解决方案中添加一个新的名为Common的类库。你可以在这个库中定义数据模型和服务协定，并引用 System.ServiceModel。

3. 修改Common工程的属性，设置目标框架为.Net Framwork 4.5.1, 目标平台为x64。

4. 在Common类库中增加一个名为*ShoppingCartItem*的类，用于定义购物车中的一个订单项的结构：

    	public struct ShoppingCartItem
    	{
    		public string ProductName { get; set; }
    		public double UnitPrice { get; set; }
    		public int Amount { get; set; }
    		public double LineTotal
    		{
    			get
    			{
    				return Amount * UnitPrice;
    			}
    		}
    	}

5. 定义服务协定，包括以下方法：一个*Add*方法用于往购物车中添加一个项，一个*Delete*方法用于从购物车中删除一项以及一个*GetItem*方法用于罗列购物车中所有的订单：

	    [ServiceContract]
	    public interface IShoppingCartService
	    {
	    	[OperationContract]
	    	Task AddItem(ShoppingCartItem item);
	    	[OperationContract]
	    	Task DeleteItem(string productName);
	    	[OperationContract]
	    	Task<List<ShoppingCartItem>> GetItems();
	    }

6. 回到*ShoppingCartService*中添加Common的工程引用。

7. 添加一个私有方法，如WCF communication stack实例中一样绑定WCF：

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

8. 简单地实现WCF communication stack：

		protected override IEnumerable<ServiceReplicaListener> CreateServiceReplicaListeners()
		{
		    return new[]
		    {
		        new ServiceReplicaListener(initParams =>
		            new WcfCommunicationListener(initParams, typeof (IShoppingCartService), this)
		            {
		                EndpointResourceName = "ServiceEndpoint",
		                Binding = this.CreateListenBinding()
		            })
		    };
		}

9. 现在实现接口*IShoppingCartService*， 添加*AddItem*方法：


		public async Task AddItem(ShoppingCartItem item)
		{
		    var cart = await this.StateManager.GetOrAddAsync<IReliableDictionary<string, ShoppingCartItem>>("myCart");
		    using (var tx = this.StateManager.CreateTransaction())
		    {
		        await cart.AddOrUpdateAsync(tx, item.ProductName, item, (k, v) => item);
		        await tx.CommitAsync();
		    }
		}

程序第一行显示了如何创建一个可靠集合实例。因为可靠数据集是多服务，不同进程甚至不同机器共享的数据结构，你必须以某种方式通知其他服务实例进程你创建了一个新的可靠数据集。而由*StatefulServiceBase*提供的*StateManager*类就是用于此，本例中使用了*StateManager*以及字符串命名标识来定位或者创建一个可靠字典集。

*StateManager*同时也提供了事务管理能力。如本例中所示，你可以针对一个或者多个可靠数据集创建一个包含一个或者多个操作的事务，而Service Fabric保证了这些操作的原子性。所有的操作在调用*CommitAsync*之后生效，如果异常抛出或者明确的*Abort*调用，那么操作将被回滚。

10.*DeleteItem*方法也一样：

		public async Task DeleteItem(string productName)
		{
		    var cart = await this.StateManager.GetOrAddAsync<IReliableDictionary<string, ShoppingCartItem>>("myCart");
		    using (var tx = this.StateManager.CreateTransaction())
		    {
		        var existing = await cart.TryGetValueAsync(tx, productName);
		        if (existing.HasValue)
		            await cart.TryRemoveAsync(tx, productName);
		        await tx.CommitAsync();
		    }
		}


11.*GetItems*方法将枚举这个集合并返回购物车中的订单列表。尽管在这个过程中没有更新操作，我们还是使用事务来隔离操作过程中其他事务可能生效的变化。基本上，这个方法将在事务开始时返回购物车的快照。

	public async Task<List<ShoppingCartItem>> GetItems()
	{
	    var cart = await this.StateManager.GetOrAddAsync<IReliableDictionary<string, ShoppingCartItem>>("myCart");
	
	    using (var tx = this.StateManager.CreateTransaction())
	    {
	        var ret = from t in cart
	                    select t.Value;
	        return ret.ToList();
	    }
	}


12.将此应用发布到本地集群。

### 简单的商店客户端

现在，我们要开发客户端程序。我们将创建一个简单的与前例相似的基于WCF的客户端程序。

1. 在解决方案中添加一个控制台应用程序，名为SimpleStoreClient。

2. 引用NuGet包*Microsoft.ServiceFabric.Services*。

3. 将控制台应用程序的目标平台从Any CPU更改为x64，更改项目的目标框架为.NET Framework 4.5.1。

4. 添加Common的工程引用。

5. 添加一个Client类型：


		public class Client:
		ServicePartitionClient<WcfCommunicationClient<IShoppingCartService>>,
		IShoppingCartService
		{
		    public Client(WcfCommunicationClientFactory<IShoppingCartService> clientFactory, Uri
		serviceName)
		            : base(clientFactory, serviceName, 1)
		    {
		    }
		
		    public Task AddItem(ShoppingCartItem item)
		    {
		        return this.InvokeWithRetryAsync(client => client.Channel.AddItem(item));
		    }
		
		    public Task DeleteItem(string productName)
		    {
		        return this.InvokeWithRetryAsync(client => client.Channel.DeleteItem(productName));
		    }
		
		    public Task<List<ShoppingCartItem>> GetItems()
		    {
		        return this.InvokeWithRetryAsync(client => client.Channel.GetItems());
		    }
		}

这段代码与前例类似。一个值得注意的是该类型的构造函增加了分区ID的入参，调用了基类构造函数不一样的重载，目前这个参数暂时写死为1。我们将在此实例后对分区进行讨论。

6.更改*Program*类，添加*CreateClientConnectionBinding*：

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

7.修改*Main*函数，往购物车中添加一个订单并读回购物车中的内容：

	static void Main(string[] args)
	{
	    Uri ServiceName = new Uri("fabric:/SimpleStoreApplication/ShoppingCartService");
	    ServicePartitionResolver serviceResolver = new ServicePartitionResolver(() => new FabricClient());
	    NetTcpBinding binding = CreateClientConnectionBinding();
	    Client shoppingClient = new Client(new WcfCommunicationClientFactory<IShoppingCartService>(serviceResolver, binding, null), ServiceName);
	    shoppingClient.AddItem(new ShoppingCartItem
	    {
	        ProductName = "XBOX ONE",
	        UnitPrice = 329.0,
	        Amount = 2
	    }).Wait();
	    var list = shoppingClient.GetItems().Result;
	    foreach(var item in list)
	    {
	        Console.WriteLine(string.Format("{0}: {1:C2} X {2} = {3:C2}",
	            item.ProductName,
	            item.UnitPrice,
	            item.Amount,
	            item.LineTotal));
	    }
	    Console.ReadKey();
	}

8.运行client程序，你将看到如下输出：

XBOX ONE: $329.00 X 2 = $658.00

### 服务分区

在第一章中介绍过，为了可扩容性，一个Service Fabric服务可以分为多个分区。在之前的无状态服务实例中，使用了单例分区，在以下应用资源配置文件中定义：

	<StatelessService ServiceTypeName="...">
	  <SingletonPartition />
	</StatelessService>

Service Fabric支持两种分区方案：统一的64位整数分区(UniformInt64Partition)和命名分区(NamedPartition)。

### UniformInt64Partition

UniformInt64Partition将连续的键值范围均匀地分配到所指定的分区。下述应用资源配置文件指定了要将键值范围（由LowKey和HighKey定义）分配到10个分区（由PartitionCount定义）。

	<StatefulService ServiceTypeName="ShoppingCartServiceType" TargetReplicaSetSize="3" MinReplicaSetSize="3">
	  <UniformInt64Partition PartitionCount="10" LowKey="1" HighKey="100" />
	</StatefulService>

然后，客户端可以通过在键值范围内指定键值来解析到特定分区：1到10解析到第一个分区，11到20解析到第二个分区，依此类推。 如果打开商店应用程序的资源配置文件，则可以看到默认情况下分区方案将一个较大键值范围（Int64.MaxValue - Int64.MinValue）放入单个分区。（为了清楚起见，我已经用数字替换了参数引用。）

	<StatefulService ServiceTypeName="ShoppingCartServiceType" TargetReplicaSetSize="3"
	MinReplicaSetSize="3">
	  <UniformInt64Partition PartitionCount="1" LowKey="-9223372036854775808"
	HighKey="9223372036854775807" />
	</StatefulService>

回想一下，在上面的例子中，当调用基础构造函数时，Client类总是传入1作为分区ID：

	public Client(WcfCommunicationClientFactory<IShoppingCartService> clientFactory, Uri
	serviceName)
	        : base(clientFactory, serviceName, 1)
	{
	}

以哪个64位整数值作为参数传递都不重要，因为所有的键都对于在同一个分区中。 但是，如果增加分区计数，则可以通过不同的分区键将客户端分布在分区之间。 如果客户端不关心哪个分区，它可以在分区键范围内生成一个随机数，则它将有一个均匀的机会路由到任何分区上。 实际上，客户端将使用某种哈希算法来实现不同的分区键分发。

作为练习，修改应用程序以使用向每个客户分发一个分区的简单分区方案。

1. 修改应用程序资源配置文件，分区个数为10，分区键的范围设定为从0到9。换句话说，服务旨在支持10个客户。 每个客户都由客户ID从0到9标识，每个客户将拥有自己的分区。拥有大量分区可能并不可取的，特别是当分区范围稀疏（由于客户数量低）时。 但是，由于无法更改分区数或正在运行的服务的分区方案类型，因此需要定义足够的分区，以便在需要时扩展服务。例如，如果有1,000个分区，并且集群有10个节点，则每个节点将获得100个分区。 如果节点的工作负担过重，可以将集群扩展为100个节点，从而将每个节点上的分区分解为10个。但是，如果开始时仅有10个分区，那么增加集群也于事无补，因为增加的节点无法被利用上。
	
		<Service Name="ShoppingCartService">
		    <StatefulService ServiceTypeName="ShoppingCartServiceType" TargetReplicaSetSize="3"	MinReplicaSetSize="2">
		      <UniformInt64Partition PartitionCount="10" LowKey="0" HighKey="9" />
		    </StatefulService>
		</Service>

局部环境约束

当您使用本地环境时，请勿定义太多的分区。 默认日志记录配置可能会对磁盘造成太大的负担。 如果错误地部署了大量的分区，则需要重启群集才能将计算机恢复到工作状态。 有关使用PowerShell进行诊断和集群管理的详细信息，请参阅第9章和第10章。

2.在SimpleStoreClient项目中，修改客户端构造函数以接收并使用客户ID（从0到9）作为分区键。

	public Client(WcfCommunicationClientFactory<IShoppingCartService> clientFactory, Uri
	serviceName, long customerId)
	        : base(clientFactory, serviceName, customerId)
	{
	}

3.在同一个项目中，向Program类添加一个新的静态方法：

	private static void PrintPartition(Client client)
	{
	    ResolvedServicePartition partition;
	    if (client.TryGetLastResolvedServicePartition(out partition))
	    {
	        Console.WriteLine("Partition ID: " + partition.Info.Id);
	    }
	}

此方法获取最后解析的分区的信息。 我们将使用此方法来显示已解析的分区标识，以检查与客户端会话的分区。

4.修改Main方法以使用不同的客户ID来调用服务：

	for (int i = 0; i < 10; i++)
	{
		Client shoppingClient = new Client(new WcfCommunicationClientFactory<IShoppingCartService>(serviceResolver, binding, null), ServiceName, i);
	    shoppingClient.AddItem(new ShoppingCartItem
	    {
	        ProductName = "XBOX ONE (" + i.ToString() + ")",
	        UnitPrice = 329.0,
	        Amount = 2
	    }).Wait();
	    shoppingClient.AddItem(new ShoppingCartItem
	    {
	        ProductName = "Halo 5 (" + i.ToString() + ")",
	        UnitPrice = 59.99,
	        Amount = 1
	    }).Wait();
	    PrintPartition(shoppingClient);
	    var list = shoppingClient.GetItems().Result;
	    foreach (var item in list)
	    {
	        Console.WriteLine(string.Format("{0}: {1:C2} X {2} = {3:C2}",
	            item.ProductName,
	            item.UnitPrice,
	            item.Amount,
	            item.LineTotal));
	    }
	}

上述代码在图3-2中生成以下输出（部分显示）。 你可以看到每个客户根据其客户ID路由到自己的分区。

![](/images/chapter3/Image1.jpg)

图3-2 修改过的Simple Store客户端的输出

在第12章中，我们将基于Actor模式，实现一个不同的购物车应用程序设计方案。

### NamedPartition

使用命名分区显式定义分区。 在某些情况下，分区的数量预知，并随时间保持静态。 例如，由美国各个州划分分区的服务将随着时间推移而具有稳定的分区。

作为练习，修改您的服务以使用命名分区。

1.修改应用程序资源配置文件以定义三个客户分区。 在现实生活中，您不必为每个客户明确定义客户分区。 此练习仅用于熟悉命名分区语法。

	<Service Name="ShoppingCartService">
	      <StatefulService ServiceTypeName="ShoppingCartServiceType" TargetReplicaSetSize="3" MinReplicaSetSize="2">
	      <NamedPartition>
	          <Partition Name="Customer 1" />
	          <Partition Name="Customer 2" />
	          <Partition Name="Customer 3" />
	      </NamedPartition>
	    </StatefulService>
	</Service>

2.在SimpleStoreClient项目中，修改客户端构造函数以使用不同的重载，增加一个字符串分区键的入参：

	public Client(WcfCommunicationClientFactory<IShoppingCartService> clientFactory, Uri serviceName, string customerId)
	        : base(clientFactory, serviceName, customerId)
	{
	}

3.在程序*Program*类中，修改循环以测试客户1,2和3.请注意在这种情况下如何使用字符串分区名称。

	for (int i = 1; i <= 3; i++)
	{
	    Client shoppingClient = new Client(new WcfCommunicationClientFactory<IShoppingCartService>
	          (serviceResolver, binding, null),
	            ServiceName, "Customer " + i);
	...

4.部署服务并运行客户端测试程序。 输出应与之前的输出类似。

## 分区和副本

介绍完实际应用场景下的分区使用，我们来介绍分区和副本。

### 副本的角色

在任何时间，一个服务副本只可能是以下几种角色（或者状态之一）：主副本，活动从副本，空闲从副本或者空。逻辑上来讲，还有一种未知状态，就是在创建副本时的一种临时为空的状态。一旦启动，副本就不会再回到未知状态。

分区的副本形成了一个副本集。所有副本集中的副本都承担以下角色之一：

- **主副本** 所有写入操作的主要副本，每个副本集每次最多只能有一个主副本。写入操作必须要在副本集中核定数量的副本确认后方可生效。对于想了解底层Paxos算法感兴趣的读者，请参阅论文“Part-Time Parliament”。

（Lamport, Leslie. “The Part-Time Parliament.” ACM Transactions on Computer Systems 16, 2 (1998): 133-169）。

- **活动从副本** 参与副本集的写入仲裁。副本集中可以有很多的从副本。当写入操作发生时，参与写入仲裁的活动从副本需要从主副本上获取和更新状态，并确认写入操作。

- **空闲从副本** 接收主副本上所有的状态更新，并准备好成为活动从副本。空闲从副本不参与写入仲裁。

- **空副本** 表示了从副本集中退出的副本。空副本没有任何状态。

图3-3描述了副本在不同角色（状态）之间的转换。

![](/images/chapter3/Image2.jpg)

图3-3 副本状态转换

要检查副本状态，可以使用Service Fabric Explorer。 在树视图中，展开分区会显示其副本及其角色，如图3-4所示。

![](/images/chapter3/Image3.jpg)

图3-4 分区和副本

如果单击每个副本，可以看到只有主副本绑定到侦听地址，表示所有读取和写入都将通过主副本，如图3-5所示。

![](/images/chapter3/Image4.jpg)

图3-5 主副本Essentials

### 可扩展性

当需要的时候可以在节点间移动副本。此功能允许Service Fabric通过重新定位副本来提供故障切换和扩展。当节点发生故障时，Service Fabric可以将副本移动到健康的节点，以保持服务的连续性。 当群集上的更多节点可用时，Service Fabric可以在所有可用节点之间重新分配副本以平衡工作负载。

为了说明扩展的过程，假设我们有四个分区，每个分区有三个副本。 这些副本分布在三个节点上，如图3-6所示。

![](/images/chapter3/Image5.jpg)

图3-6 三节点集群上的副本分布

假设另一个节点被添加到集群中。 Service Fabric会一直从集群中的所有节点获取负载信息。 当它检测到额外的节点可用于承担更多的工作负载时，它将尝试将副本重新定位来平均所有节点上的工作负载。（第6章和第7章提供了有关此机制的更多详细信息。）图3-7显示了这种重新分配的结果。 本图中，节点4被添加到集群中，一些副本移动到新节点4以均衡节点上的工作负载。

![](/images/chapter3/Image6.jpg)

图3-7 四节点集群上的副本分布

## 附加信息

如果您对Paxos感兴趣，除了参考书之外，您可以阅读Leslie Lamport的“Paxos Made Simple”论文：http://research.microsoft.com/en-us/um/people/lamport/pubs/ paxos-simple.pdf。