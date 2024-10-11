# 微服务开源框架 TARS 的 RPC 源码解析 之 初识 TARS C++ 客户端

![banner](E:\MarkDown\tars框架\picture\watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RBUlNGb3VuZGF0aW9u,size_16,color_FFFFFF,t_70.jpeg)

作者：Cony 导语：微服务开源框架 TARS 的 RPC 调用包含客户端与服务端，《微服务开源框架 TARS 的 RPC 源码解析》系列文章将从初识客户端、客户端的同步及异步调用、初识服务端、服务端的工作流程四部分，以 C++ 语言为载体，深入浅出地带你了解 TARS RPC 调用的原理。

## 什么是 TARS

TARS 是腾讯使用十年的微服务开发框架，目前支持 C++、Java、PHP、Node.js、Go 语言。该开源项目为用户提供了涉及到开发、运维、以及测试的一整套微服务平台 PaaS 解决方案，帮助一个产品或者服务快速开发、部署、测试、上线。目前该框架应用在腾讯各大核心业务，基于该框架部署运行的服务节点规模达到数十万。 TARS 的通信模型中包含客户端和服务端。客户端服务端之间主要是利用 RPC 进行通信。本系列文章分上下两篇，对 RPC 调用部分进行源码解析。本文是上篇，我们将以 C++ 语言为载体，带大家了解一下 TARS 的客户端。

## 初识客户端

TARS 的客户端最重要的类是 Communicator，一个客户端只能声明出一个 Communicator 类实例，用户可以通过 CommunicatorPtr& Application::getCommunicator () 获取线程安全的 Communicator 类单例。Communicator 类聚合了两个比较重要的类，一个是 CommunicatorEpoll，负责网络线程的建立与通过 ObjectProxyFactory 生成 ObjectProxy；另一个是 ServantProxyFactory，生成不同的 RPC 服务句柄，即 ServantProxy，用户通过 ServantProxy 调用 RPC 服务。下面简单介绍几个类的作用。

### Communicator

一个 Communicator **实例就是一个客户端，负责与服务端建立连接**，生成 RPC 服务句柄，可以通过 CommunicatorPtr& Application::getCommunicator () 获取 Communicator 实例，用户最后不要自己声明定义新的 Communicator 实例。

### ServantProxy 与 ServantProxyFactory

ServantProxy 就是一个**服务代理**，ServantProxy 可以通过 ServantProxyFactory 工厂类生成，用户往往通过 Communicator 的 template<class T> void stringToProxy () 接口间接调用 ServantProxyFactory 的 ServantPrx::element_type* getServantProxy () 接口以获取服务代理，通过服务代理 ServantProxy，用户就可以进行 RPC 调用了。ServantProxy 内含多个服务实体 ObjectProxy（详见下文第 4 小点），能够帮助用户在同一个服务代理内进行负载均衡。

### CommunicatorEpoll

CommunicatorEpoll 类代表客户端的**网络模块**，内含 TC_Epoller 作为 IO 复用，能够同时处理不同主调线程（caller 线程）的多个请求。CommunicatorEpoll 内含服务实体工厂类 ObjectProxyFactory（详见下文第 4 小点），意味着在同一网络线程中，能够产生不同服务的实体，能够完成不同的 RPC 服务调用。CommunicatorEpoll 还聚合了异步调用处理线程 AsyncProcThread，负责接收到异步的响应包之后，将响应包交给该线程处理。

### ObjectProxy 与 ObjectProxyFactory

ObjectProxy 类是一个**服务实体**，注意与 ServantProxy 类是一个服务代理相区别，前者表示一个网络线程上的某个服务实体 A，后者表示对所有网络线程上的某服务实体 A 的总代理，更详细的介绍可见下文。ObjectProxy 通过 ObjectProxyFactory 生成，而 ObjectProxyFactory 类的实例是 CommunicatorEpoll 的成员变量，意味着一个网络线程 CommunicatorEpoll 能够产生各种各样的服务实体 ObjectProxy，发起不同的 RPC 服务。ObjectProxy 通过 AdapterProxy 来管理对服务端的连接。 好了，介绍完所有的类之后，先通过类图理一理他们之间的关系，这个类图在之后的文章中将会再次出现。

![图（1-1）客户端类图](E:\MarkDown\tars框架\picture\up-ed4ee7ab0ac954aeb21ff13edaf5b8db11b.png)

TARS 的客户端最重要的类是 Communicator，一个客户端**只能声明出一个 Communicator 类实例**，用户可以通过 CommunicatorPtr& Application::getCommunicator () 获取线程安全的 Communicator 类单例。Communicator 类聚合了两个比较重要的类，一个是 CommunicatorEpoll，负责网络线程的建立与通过 ObjectProxyFactory 生成 ObjectProxy；另一个是 ServantProxyFactory，生成不同的 RPC 服务句柄，即 ServantProxy，用户通过 ServantProxy 调用 RPC 服务。 根据用户配置，Communicator 拥有 **n 个网络线程**，**即 n 个 CommunicatorEpoll**。每个 CommunicatorEpoll 拥有一个 ObjectProxyFactory 类，每个 ObjectProxyFactory 可以生成一系列的不同服务的实体对象 ObjectProxy，因此，假如 Communicator 拥有两个 CommunicatorEpoll，并有 foo 与 bar 这两类不同的服务实体对象，那么如下图（1-2）所示，每个 CommunicatorEpoll 可以通过 ObjectProxyFactory 创建两类 ObjectProxy，这是 TARS 客户端的第一层**负载均衡**，每个线程都可以分担所有服务的 RPC 请求，因此，一个服务的阻塞可能会影响其他服务，因为网络线程是多个服务实体 ObjectProxy 所共享的。

![图（1-2）Communicator中的CommunicatorEpoll与ObjectProxy](E:\MarkDown\tars框架\picture\up-2ec6ba99f8ffe019386dc4e2c3f4e03c798.png)

Communicator 类下另一个比较重要的 ServantProxyFactory 类的作用是依据实际服务端的信息（如服务器的 socket 标志）与 Communicator 中客户端的信息（如网络线程数）而**生成 ServantProxy 句柄，通过句柄调用 RPC 服务**。举个例子，如下图（1-3）所示，Communicator 实例通过 ServantProxyFactory 成员变量的 getServantProxy () 接口在构造 fooServantProxy 句柄的时候，会获取 Communicator 实例下的所有 CommunicatorEpoll（即 CommunicatorEpoll-1 与 CommunicatorEpoll-2）中的 fooObjectProxy（即 fooObjectProxy-1 与 fooObjectProxy-2），并作为构造 fooServantProxy 的参数。Communicator 通过 ServantProxyFactory 能够获取 foo 与 bar 这两类 ServantProxy，ServantProxy 与相应的 ObjectProxy 存在相应的聚合关系：

![图（1-3）Communicator中的ServantProxyFactory与ObjectProxy](E:\MarkDown\tars框架\picture\up-244f93ace1537515b83b824cbb84af637cf.png)

另外，每个 ObjectProxy 都拥有一个 EndpointManager，例如，fooObjectProxy 的 **EndpointManager 管理 fooObjectProxy 下面的所有 fooAdapterProxy**，每个 AdapterProxy 连接到一个提供相应 foo 服务的服务端物理机 socket 上。通过 EndpointManager 还可以以不同的负载均衡方式获取连接 AdapterProxy。假如 foo 服务有两台物理机，bar 服务有一台物理机，那么 ObjectProxy，EndpointManager 与 AdapterProxy 关系如下图（1-4）所示。上面提到，不同的网络线程 **CommunicatorEpoll 均可以发起同一 RPC 请求，对于同一 RPC 服务，选取不同的 ObjectProxy**（或可认为选取不同的网络线程 CommunicatorEpoll）是**第一层的负载均衡**，而对于同一个被选中的 **ObjectProxy，通过 EndpointManager 选择不同的 socket 连接 AdapterProxy**（假如 ObjectProxy 有大于 1 个的 AdapterProxy，如图（1-4）的 fooObjectProxy）是**第二层的负载均衡**。

![图（1-4）ObjectProxy与AdapterProxy的关系](E:\MarkDown\tars框架\picture\up-71479b7e4a022ae1d3ed1533276bd65e2d2.png)

在客户端进行初始化时，必须建立上面介绍的关系，因此相应的类图如图（1-5）所示，通过类图可以看出各类的关系，以及初始化需要用到的函数。

![图（1-5）客户端初始化后建立的类图](E:\MarkDown\tars框架\picture\up-3925cbef9b6c357db448a3c53b902c27dfc.png)

## 初始化代码

现在，通过代码跟踪来看看，在客户端初始化过程中，各个类是如何被初始化出来并建立上述的架构关系的。在简述之前，可以先看看函数的调用流程图，若看不清晰，可以将图片保存下来，用看图软件放大查看，强烈建议结合文章的代码解析以 TARS 源码一起查看，文章后面的所有代码流程图均如此。 接下来，将会按照函数调用流程图来一步一步分析客户端代理是如何被初始化出来的：

![图（1-6） 初始化函数调用过程图](E:\MarkDown\tars框架\picture\up-d8b47ad48dbe5b1ab9c5fa36755e6f3e0f3.png)

### 1. 执行 stringToProxy

在客户端程序中，一开始会执行下面的代码进行整个客户端代理的初始化：

```c++
Communicator comm;
HelloPrx prx;
comm.stringToProxy("TestApp.HelloServer.HelloObj@tcp -h 1.1.1.1 -p 20001" , prx);
```

先声明一个 Communicator 变量 comm（其实不建议这么做）以及一个 ServantProxy 类的指针变量 prx，在此处，服务为 Hello，因此声明一个 HelloPrx prx。注意一个客户端只能拥有一个 Communicator。为了能够获得 RPC 的服务句柄，我们调用 Communicator::stringToProxy ()，并传入服务端的信息与 prx 变量，函数返回后，prx 就是 RPC 服务的句柄。 进入 Communicator::stringToProxy () 函数中，我们通过 Communicator::getServantProxy () 来依据 objectName 与 setName 获取服务代理 ServantProxy：

```c++
/**
* 生成代理
* @param T
* @param objectName
* @param setName 指定set调用的setid
* @param proxy
*/
template<class T> void stringToProxy(const string& objectName, T& proxy,const string& setName="")
{
    ServantProxy * pServantProxy = getServantProxy(objectName,setName);
    proxy = (typename T::element_type*)(pServantProxy);
}
```

### 2. 执行 Communicator 的初始化函数

进入 Communicator::getServantProxy ()，首先会执行 Communicator::initialize () 来初始化 Communicator，需要注意一点，Communicator:: initialize () 只会被执行一次，下一次执行 Communicator::getServantProxy () 将不会再次执行 Communicator:: initialize () 函数：

```cpp
void Communicator::initialize()
{
    TC_LockT<TC_ThreadRecMutex> lock(*this);
    if (_initialized) //已经被初始化则直接返回
        return;
    ......
｝
```

进入 Communicator::initialize () 函数中，在这里，将会 new 出上文介绍的与 Communicator 密切相关的类 ServantProxyFactory 与 n 个 CommunicatorEpoll，n 为客户端的网络线程数，最小为 1，最大为 MAX_CLIENT_THREAD_NUM：

```cpp
void Communicator::initialize()
{
    ......
    _servantProxyFactory = new ServantProxyFactory(this);
    ......
    for(size_t i = 0; i < _clientThreadNum; ++i)
    {
        _communicatorEpoll[i] = new CommunicatorEpoll(this, i);
        _communicatorEpoll[i]->start(); //启动网络线程
    }
    ......
｝
```

在 CommunicatorEpoll 的构造函数中，ObjectProxyFactory 被创建出来，这是构造图（1-2）关系的前提。除此之外，还可以看到获取相应配置，创建并启动若干个异步回调后的处理线程。创建完成后，调用 CommunicatorEpoll::start () 启动网络线程。至此，Communicator::initialize () 顺利执行。通过下图回顾上面的过程：

![图（1-7）执行Communicator的初始化函数流程](E:\MarkDown\tars框架\picture\up-b937cc166174ac404e7faa3908f86b1450b.png)

### 3. 尝试获取 ServantProxy

代码回到 Communicator::getServantProxy () 中 Communicator::getServantProxy () 会执行 ServantProxyFactory::getServantProxy () 并返回相应的服务代理：

```cpp
ServantProxy* Communicator::getServantProxy(const string& objectName, const string& setName)
{
    ……
    return _servantProxyFactory->getServantProxy(objectName,setName);
}
```

进入 ServantProxyFactory::getServantProxy ()，首先会加锁，从 map<string, ServantPrx> _servantProxy 中查找目标，若查找成功直接返回。若查找失败，TARS 需要构造出相应的 ServantProxy，ServantProxy 的构造需要如图（1-3）所示的相对应的 ObjectProxy 作为构造函数的参数，由此可见，在 ServantProxyFactory::getServantProxy () 中有如下获取 ObjectProxy 指针数组的代码：

```cpp
ObjectProxy ** ppObjectProxy = new ObjectProxy * [_comm->getClientThreadNum()];
assert(ppObjectProxy != NULL);
for(size_t i = 0; i < _comm->getClientThreadNum(); ++i)
{
    ppObjectProxy[i] = _comm->getCommunicatorEpoll(i)->getObjectProxy(name, setName);
}
```

### 4. 获取 ObjectProxy

代码来到 ObjectProxyFactory::getObjectProxy ()，同样，会首先加锁，再从 map<string,ObjectProxy*> _objectProxys 中查找是否已经拥有目标 ObjectProxy，若查找成功直接返回。若查找失败，需要新建一个新的 ObjectProxy，通过类图可知，ObjectProxy 需要一个 CommunicatorEpoll 对象进行初始化，由此关联管理自己的 CommunicatorEpoll，CommunicatorEpoll 之后便可以通过 getObjectProxy () 接口获取属于自己的 ObjectProxy。详细过程可见下图：

![图（1-8）获取ObjectProxy流程](E:\MarkDown\tars框架\picture\up-a34f16584e5c81dd293118013ddfebef295.png)

注意：pObjectProxy->initialize放到getServantProxy里的tars_initialize统一初始化。ObjectProxy创建的过程，被包含在了createObjectProxy函数中，

### 5. 建立 ObjectProxy 与 AdapterProxy 的关系

新建 ObjectProxy 的过程同样非常值得关注，在 ObjectProxy::ObjectProxy () 中，关键代码是：

```cpp
_endpointManger.reset(new EndpointManager(this, _communicatorEpoll->getCommunicator(), sObjectProxyName, pCommunicatorEpoll->isFirstNetThread(), setName));
```

每个 ObjectProxy 都有属于自己的 EndpointManager 负责管理到服务端的所有 socket 连接 AdapterProxy，每个 AdapterProxy 连接到一个提供相应服务的服务端物理机 socket 上。通过 EndpointManager 还可以以不同的负载均衡方式获取与服务器的 socket 连接 AdapterProxy。 ObjectProxy:: ObjectProxy () 是图（1-6）或者图（1-8）中的略 1，具体的代码流程如图（1-9）所示。ObjectProxy 创建一个 EndpointManager 对象，在 EndpointManager 的初始化过程中，依据客户端提供的信息，直接创建连接到服务端物理机的 TCP/UDP 连接 AdapterProxy 或者从代理中获取服务端物理机 socket 列表后再创建 TCP/UDP 连接AdapterProxy。

![图（1-9）ObjectProxy::ObjectProxy()函数流程图](E:\MarkDown\tars框架\picture\up-bcb4775a58488fe52590099b9d36681be90.png)

按照图（1-9）的程序流程执行完成后，便会建立如图（2-3）所示的一个 ObjectProxy 对多个 AdapterProxy 的关系。 新建 ObjectProxy 之后，就可以调用其 ObjectProxy::initialize () 函数进行 ObjectProxy 对象的初始化了，当然，需要将 ObjectProxy 对象插入 ObjectProxyFactory 的成员变量_objectProxys 与_vObjectProxys 中，方便下次直接返回 ObjectProxy 对象。

注意：_endpointManger是在ObjectProxy::initialize里

==注意：初始化代码有所变化、注意判断==

### 6. 继续完成 ServantProxy 的创建

退出层层的函数调用栈，代码再次回 ServantProxyFactory::getServantProxy ()，此时，ServantProxyFactory 已经获得相应的 ObjectProxy 数组 ObjectProxy** ppObjectProxy，接着便可以调用：

```cpp
ServantPrx sp = new ServantProxy(_comm, ppObjectProxy, _comm->getClientThreadNum());
```

进行 ServantProxy 的构造。构造完成便可以呈现出如图（2-1）的关系。在 ServantProxy 的构造函数中可以看到，ServantProxy 在新建一个 EndpointManagerThread 变量，这是对外获取路由请求的类，是 TARS 为调用逻辑而提供的多种解决跨地区调用等问题的方案。同时可以看到：

```cpp
for(size_t i = 0;i < _objectProxyNum; ++i)
{
   (*(_objectProxy + i))->setServantProxy(this);
}
```

建立了 ServantProxy 与 ObjectProxy 的相互关联关系。剩下的是读取配置文件，获取相应的信息。 构造 ServantProxy 变量完成后，ServantProxyFactory::getServantProxy () 获取一些超时参数，赋值给 ServantProxy 变量，同时将其放进 map<string, ServantPrx> _servantProxy 中，方便下次直接查找获取。 ServantProxyFactory::getServantProxy () 将刚刚构造的 ServantProxy 指针变量返回给调用他的 Communicator::getServantProxy ()，在 Communicator::getServantProxy () 中：

```cpp
ServantProxy * Communicator::getServantProxy(const string& objectName,const string& setName)
{
    ……
    return _servantProxyFactory->getServantProxy(objectName,setName);
}
```

直接将返回值返回给调用起 Communicator::getServantProxy () 的 Communicator::stringToProxy ()。可以看到：

```cpp
template<class T> void stringToProxy(const string& objectName, T& proxy,const string& setName="")
{
    ServantProxy * pServantProxy = getServantProxy(objectName,setName);
    proxy = (typename T::element_type*)(pServantProxy);
}
```

Communicator::stringToProxy () 将返回值强制转换为客户端代码中与 HelloPrx prx 同样的类型 HelloPrx。由于函数参数 proxy 就是 prx 的引用。那么实际就是将句柄 prx 成功初始化了，用户可以利用句柄 prx 进行 RPC 调用了。

## 同步调用

当我们获得一个 **ServantProxy 句柄后，便可以进行 RPC 调用了**。Tars 提供四种 RPC 调用方式，分别是同步调用，异步调用，promise 调用与协程调用。其中最简单最常见的 RPC 调用方式是同步调用，接下来，将简单分析 Tars 的同步调用。

现假设有一个 MyDemo.StringServer.StringServantObj 的服务，提供一个 RPC 接口是 append，传入两个 string 类型的变量，返回两个 string 类型变量的拼接结果。而且假设有两台服务器，socket 标识分别是 192.168.106.129:34132 与 192.168.106.130:34132，设置客户端的网络线程数为 3，那么执行如下代码：

```cpp
Communicator _comm;
StringServantPrx _proxy;
_comm.stringToProxy("MyDemo.StringServer.StringServantObj@tcp -h 192.168.106.129 -p 34132", _proxy);
_comm.stringToProxy("MyDemo.StringServer.StringServantObj@tcp -h 192.168.106.130 -p 34132", _proxy);
```

经过上文关于客户端初始化的分析介绍可知，可以得出如下图所示的关系图：

![图（1-10）StringServant中ServantProxy，ObjectProxy与AdapterProxy的关系](E:\MarkDown\tars框架\picture\up-ae1404c6bd8bce139316ac39df071bfc212.png)

获取 StringServantPrx _proxy 后，直接调用：

```cpp
string str1(abc-), str2(defg), rStr;
int retCode = _proxy->append(str1, str2, rStr);
```

成功进行 RPC 同步调用后，返回的结果是 rStr = “abc-defg”。

同样，我们先看看与同步调用相关的类图，如下图所示：

![图（1-11） 客户端同步调用涉及的类图](E:\MarkDown\tars框架\picture\up-8bd79f22109b109513281ad9149d31918b7.png)

StringServantProxy 是继承自 ServantProxy 的，StringServantProxy 提供了 RPC 同步调用的接口 Int32 append ()，当用户发起同步调用_proxy->append (str1, str2, rStr) 时，所进行的函数调用过程如下图所示。

![图（1-12）同步调用过程](E:\MarkDown\tars框架\picture\up-295195bb9a5120e121b6b1f3df163f2ff17.png)

在函数 StringServantProxy::append () 中，程序会先构造 ServantProxy::tars_invoke () 所需要的参数，如请求包类型，RPC 方法名，方法参数等，需要值得注意的是，传递参数中有一个 **ResponsePacket 类型的变量，在同步调用中，最终的返回结果会放置在这个变量上**。接下来便直接调用了 ServantProxy::tars_invoke () 方法：

```cpp
tars_invoke(tars::TARSNORMAL, "append", _os.getByteBuffer(), context, _mStatus, rep);
```

在 ServantProxy::tars_invoke () 方法中，先创建一个 ReqMessage 变量 msg，初始化 msg 变量，给变量赋值，如 Tars 版本号，数据包类型，服务名，RPC 方法名，Tars 的上下文容器，同步调用的超时时间（单位为毫秒）等。最后，调用 ServantProxy::invoke () 进行远程方法调用。

无论同步调用还是各种异步调用，**ServantProxy::invoke () 都是 RPC 调用的必经之地**。在 ServantProxy::invoke () 中，继续填充传递进来的变量 ReqMessage msg。此外，还需要获取调用者 caller 线程的线程私有数据 ServantProxyThreadData，用来指导 RPC 调用。客户端的每个 caller 线程都有属于自己的维护调用上下文的线程私有数据，如 hash 属性，消息染色信息。最关键的还是每条 caller 线程与每条客户端网络线程 CommunicatorEpoll 进行信息交互的桥梁 —— 通**信队列 ReqInfoQueue 数组**，数组中的每个 ReqInfoQueue 元素负责与一条网络线程进行交互，如图（1-13）所示，图中橙色阴影代表数组 **ReqInfoQueue[]**，阴影内的圆柱体代表数组元素 **ReqInfoQueue**。假如客户端 create 两条线程（下称 caller 线程）发起 StringServant 服务的 RPC 请求，且客户端网络线程数设置为 2，那么两条 caller 线程各自有属于自己的线程私有数据**请求队列数组 ReqInfoQueue []**，数组里面的 ReqInfoQueue 元素便是该数组对应的 caller 线程与两条网络线程的通信桥梁，一条网络线程对应着数组里面的一个元素，通过网络线程 ID 进行数组索引。整个关系有点像生产者消费者模型，生产者 Caller 线程向自己的线程私有数据 **ReqInfoQueue []** 中的第 N 个元素 ReqInfoQueue [N] push 请求包，消费者客户端第 N 个网络线程就会从这个队列中 pop 请求包。·

![图（1-13）caller线程与网络线程的通信](E:\MarkDown\tars框架\picture\up-889d7d1ffb847650ce16ed39ee35e7c88e1.png)

阅读代码可能会发现几个常量值，如 MAX_CLIENT_THREAD_NUM=64，这是最大网络线程数，在图（1-13）中为单个请求队列数组 ReqInfoQueue [] 的最大 size；MAX_CLIENT_NOTIFYEVENT_NUM=2048，在图（1-13）中，可以看作 caller 线程的最大数量，或者请求队列数组 ReqInfoQueue [] 的最大数量（反正两者一一对应，每个 caller 线程都有自己的线程私有数据 ReqInfoQueue []）。

接着依据 caller 线程的线程私有数据进行第一次的负载均衡 —— 选取 **ObjectProxy（即选择网络线程 CommunicatorEpoll）和与之相对应的 ReqInfoQueue**（二者一 一对应）：

```cpp
ObjectProxy * pObjProxy = NULL;
ReqInfoQueue * pReqQ = NULL;
//选择网络线程
selectNetThreadInfo(pSptd, pObjProxy, pReqQ);
```

在 ServantProxy::selectNetThreadInfo () 中，通过轮询的形式来选取 ObjectProxy 与 ReqInfoQueue。

退出 ServantProxy::selectNetThreadInfo () 后，便得到 ObjectProxy_类型的 pObjProxy 及其对应的 ReqInfoQueue_类型的 ReqInfoQueue，稍后**通过 pObjectProxy 来发送 RPC 请求**，**请求信息会暂存在 ReqInfoQueue** 中。

由于是同步调用，需要新建一个条件变量去监听 RPC 的完成，可见：

```cpp
//同步调用 new 一个ReqMonitor
assert(msg->pMonitor == NULL);
if(msg->eType == ReqMessage::SYNC_CALL)
{
	msg->bMonitorFin = false;
	if(pSptd->_sched)
	{
    	msg->bCoroFlag = true;
    	msg->sched 	= pSptd->_sched;
    	msg->iCoroId   = pSptd->_sched->getCoroutineId();
	}
	else
	{
    	msg->pMonitor = new ReqMonitor;
	}
}
```

创建完条件变量，接下来往 ReqInfoQueue 中 push_back () 请求信息包 msg，并通知 **pObjProxy 所属的 CommunicatorEpoll 进行数据发送**： TARS C++ 客户端

```cpp
if(!pReqQ->push_back(msg,bEmpty))
{
	TLOGERROR("[TARS][ServantProxy::invoke msgQueue push_back error num:" << pSptd->_netSeq << "]" << endl);
	delete msg;
	msg = NULL;
	pObjProxy->getCommunicatorEpoll()->notify(pSptd->_reqQNo, pReqQ);
	throw TarsClientQueueException("client queue full");
}
 
pObjProxy->getCommunicatorEpoll()->notify(pSptd->_reqQNo, pReqQ);
```

来到 CommunicatorEpoll::notify () 中，往请求事件通知数组 NotifyInfo _notify [] 中添加请求事件，通知 CommunicatorEpoll 进行请求包的发送。注意了，这个函数的作用仅仅是**通知网络线程准备发送数据**，通过 TC_Epoller::mod () 或者 TC_Epoller::add () 触发一个 EPOLLIN 事件，从而促使阻塞在 TC_Epoller::wait ()（在 CommunicatorEpoll::run () 中阻塞）的网络线程 CommunicatorEpoll 被唤醒，并设置唤醒后的 epoll_event 中的联合体 epoll_data 变量为 &_notify [iSeq].stFDInfo：

```cpp
void CommunicatorEpoll::notify(size_t iSeq,ReqInfoQueue * msgQueue)
{
	assert(iSeq < MAX_CLIENT_NOTIFYEVENT_NUM);
 
	if(_notify[iSeq].bValid)
	{
    	_ep.mod(_notify[iSeq].notify.getfd(),(long long)&_notify[iSeq].stFDInfo, EPOLLIN);
    	assert(_notify[iSeq].stFDInfo.p == (void*)msgQueue);
	}
	else
	{
    	_notify[iSeq].stFDInfo.iType   = FDInfo::ET_C_NOTIFY;
    	_notify[iSeq].stFDInfo.p   	= (void*)msgQueue;
	    _notify[iSeq].stFDInfo.fd  	= _notify[iSeq].eventFd;
    	_notify[iSeq].stFDInfo.iSeq	= iSeq;
    	_notify[iSeq].notify.createSocket();
    	_notify[iSeq].bValid       	= true;
 
    	_ep.add(_notify[iSeq].notify.getfd(),(long long)&_notify[iSeq].stFDInfo, EPOLLIN);
	}
}
```

就是经过这么一个操作，网络线程就可以被唤醒，唤醒后通过 epoll_event 变量可获得 **&_notify [iSeq].stFDInfo**。接下来的请求发送与响应的接收会在后面会详细介绍。

随后，代码再次回到 ServantProxy::invoke ()，阻塞于：

```cpp
if(!msg->bMonitorFin)
{
	TC_ThreadLock::Lock lock(*(msg->pMonitor));
	//等待直到网络线程通知过来
	if(!msg->bMonitorFin)
   {
    	msg->pMonitor->wait();
	}
}
```

等待网络线程接收到数据后，对其进行唤醒。 接收到响应后，检查是否成功获取响应，是则直接退出函数即可，响应信息在传入的参数 msg 中：

```cpp
if(msg->eStatus == ReqMessage::REQ_RSP && msg->response.iRet == TARSSERVERSUCCESS)
{
	snprintf(pSptd->_szHost, sizeof(pSptd->_szHost), "%s", msg->adapter->endpoint().desc().c_str());
	//成功
	return;
}
```

若接收失败，会抛出异常，并删除 msg：

```cpp
TarsException::throwException(ret, os.str());
```

若接收成功，退出 ServantProxy::invoke () 后，回到 ServantProxy::tars_invoke ()，获取 ResponsePacket 类型的响应信息，并删除 msg 包：

```cpp
rsp = msg->response;
delete msg;
msg = NULL;
```

代码回到 StringServantProxy::append ()，此时经过同步调用，可以直接获取 RPC 返回值并回到客户端中。

### 网络线程发送请求

==注意：这又很大的不同，从run就不一样了，run->handle->handleNotify->...->AdapterProxy->invoke->invoke_connection_serial/parallel->sendRequest==

上面提到，当在 ServantProxy::invoke () 中，调用 CommunicatorEpoll::notify () 通知网络线程进行请求发送后，接下来，网络线程的具体执行流程如下图所示：

![图（1-14）网络线程发送请求包](E:\MarkDown\tars框架\picture\up-9306224ce9c039a30985ebcdb464869548b.png)

由于 CommunicatorEpoll 继承自 TC_Thread，在上文 1.2.2 节中的第 2 小点的初始化 CommunicatorEpoll 之后便调用其 CommunicatorEpoll::start () 函数启动网络线程，网络线程在 CommunicatorEpoll::run () 中一直等待_ep.wait (iTimeout)。由于在上一节的描述中，在 CommunicatorEpoll::notify ()，caller 线程发起了通知 notify，网络线程在 CommunicatorEpoll::run () 就会调用 CommunicatorEpoll::handle () 处理通知：

```cpp
void CommunicatorEpoll::run()
{
	......
    	try
    	{
        	int iTimeout = ((_waitTimeout < _timeoutCheckInterval) ? _waitTimeout : _timeoutCheckInterval);
 
        	int num = _ep.wait(iTimeout);
 
        	if(_terminate)
        	{
            	break;
        	}
 
        	//先处理epoll的网络事件
        	for (int i = 0; i < num; ++i)
        	{
            	//获取epoll_event变量的data，就是1.3.1节中提过的&_notify[iSeq].stFDInfo
            	const epoll_event& ev = _ep.get(i);
            	uint64_t data = ev.data.u64;
 
            	if(data == 0)
            	{
                	continue; //data非指针, 退出循环
            	}
            	handle((FDInfo*)data, ev.events);
        	}
    	}
	......
   
}
```

==注意：处理epoll的网络事件在TC_Epoller::done里==

在 CommunicatorEpoll::handle () 中，通过传递进来的 epoll_event 中的 data 成员变量获取前面被选中的 ObjectProxy 并调用其 ObjectProxy::invoke () 函数：

```cpp
void CommunicatorEpoll::handle(FDInfo * pFDInfo, uint32_t events)
{
	try
	{
    	assert(pFDInfo != NULL);
 
    	//队列有消息通知过来
    	if(FDInfo::ET_C_NOTIFY == pFDInfo->iType)
    	{
        	ReqInfoQueue * pInfoQueue=(ReqInfoQueue*)pFDInfo->p;
        	ReqMessage * msg = NULL;
 
        	try
        	{
            	while(pInfoQueue->pop_front(msg))
            	{
             	......
 
                	try
                	{
                    	msg->pObjectProxy->invoke(msg);
                	}
                	......
            	}
        	}
        	......
    	}
    	......
}
```

==注意：handleNotify==

在 ObjectProxy::invoke () 中将进行第二次的负载均衡，像图（1-4）所示，每个 ObjectProxy 通过 EndpointManager 可以以不同的负载均衡方式对 AdapterProxy 进行选取选择：

```cpp
void ObjectProxy::invoke(ReqMessage * msg)
{
	......
	//选择一个远程服务的Adapter来调用
	AdapterProxy * pAdapterProxy = NULL;
	bool bFirst = _endpointManger->selectAdapterProxy(msg, pAdapterProxy);
	......
}
```

在 EndpointManager:: selectAdapterProxy () 中，有多种负载均衡的方式来选取 AdapterProxy，如 getHashProxy ()，getWeightedProxy ()，getNextValidProxy () 等。

获取 AdapterProxy 之后，便将选择到的 AdapterProxy 赋值给 EndpointManager:: selectAdapterProxy () 函数中的引用参数 pAdapterProxy，随后执行：

```cpp
void ObjectProxy::invoke(ReqMessage * msg)
{
   ......
	msg->adapter = pAdapterProxy;
	pAdapterProxy->invoke(msg);
}
```

调用 pAdapterProxy 将请求信息发送出去。而在 AdapterProxy::invoke () 中，AdapterProxy 将调用 **Transceiver::sendRequset ()** 进行请求的发送。 至此，对应同步调用的网络线程发送请求的工作就结束了，网络线程会回到 CommunicatorEpoll::run () 中，继续等待数据的收发。

==注意：AdapterProxy的invoke并不是直接的调用sendRequset 函数对请求进行发送，而是串行并行发送判断，采用`invokde_connection_parallel` 和 `invokde_connection_serial`进行连接发送 ，在连接完成后，服务端返回的请求通过`onRequestCallback`里的`doInvoke（）`进行消息的返回==

### 网络线程接收响应

==注意：变化，没捋清楚==

当网络线程 CommunicatorEpoll 接收到响应数据之后，如同之前发送请求那样， 在 CommunicatorEpoll::run () 中，程序获取活跃的 epoll_event 的变量，并将其中的 epoll_data_t data 传递给 CommunicatorEpoll::handle ()：

```cpp
//先处理epoll的网络事件
for (int i = 0; i < num; ++i)
{
	const epoll_event& ev = _ep.get(i);
	uint64_t data = ev.data.u64;
 
	if(data == 0)
   {
   	continue; //data非指针, 退出循环
	}
	handle((FDInfo*)data, ev.events);
}
```

接下来的程序流程如下图所示：

![图（1-15）网络线程接收响应包](E:\MarkDown\tars框架\picture\up-37afb8a022e421c6550d19eb42b7e749908.png)

在 CommunicatorEpoll::handle () 中，从 epoll_data::data 中获取 Transceiver 指针，并调用 CommunicatorEpoll::handleInputImp ()：

```cpp
Transceiver *pTransceiver = (Transceiver*)pFDInfo->p;
//先收包
if (events & EPOLLIN)
{
	try
	{
handleInputImp(pTransceiver);
	}
	catch(exception & e)
	{
TLOGERROR("[TARS]CommunicatorEpoll::handle exp:"<<e.what()<<" ,line:"<<__LINE__<<endl);
	}
	catch(...)
	{
TLOGERROR("[TARS]CommunicatorEpoll::handle|"<<__LINE__<<endl);
	}
}
```

==👆注意：handleInputImp是通过回调调用的，不是直接调用的==

==👇注意：handleInputImp 仅负责doResponse ，而 finishInvoke 交由AdapterProxy::onRequestCallbcak结束调用==

在 CommunicatorEpoll::handleInputImp () 中，除了对连接的判断外，主要做两件事，调用 Transceiver::doResponse () 以及 AdapterProxy::finishInvoke (ResponsePacket&)，前者的工作是从 socket 连接中获取响应数据并判断接收的数据是否为一个完整的 RPC 响应包。后者的作用是将响应结果返回给客户端，同步调用的会唤醒阻塞等待在条件变量中的 caller 线程，异步调用的会在异步回调处理线程中执行回调函数。 在 AdapterProxy::finishInvoke (ResponsePacket&) 中，需要注意一点，假如是同步调用的，需要获取响应包 rsp 对应的 ReqMessage 信息，在 Tars 中，执行：

```cpp
ReqMessage * msg = NULL;
// 获取响应包rsp对应的msg信息，并在超时队列中剔除该msg
bool retErase = _timeoutQueue->erase(rsp.iRequestId, msg);
```

在找回响应包对应的请求信息 msg 的同时，将其在超时队列中剔除出来。接着执行：

```cpp
msg->eStatus = ReqMessage::REQ_RSP;
msg->response = rsp;
finishInvoke(msg);
```

==注意：对应上面发送--invokde_connection_serial、invokde_connection_parallel，结束也有专门的finishInvoke（rsp） 里也判断finishInvoke_serial、parallel==

程序调用另一个重载函数 AdapterProxy::finishInvoke (ReqMessage*)，在 AdapterProxy::finishInvoke (ReqMessage*) 中，不同的 RPC 调用方式会执行不同的动作，例如同步调用会唤醒对应的 caller 线程：

```cpp
//同步调用，唤醒ServantProxy线程
if(msg->eType == ReqMessage::SYNC_CALL)
{
	if(!msg->bCoroFlag)
	{
    	assert(msg->pMonitor);
 
    	TC_ThreadLock::Lock sync(*(msg->pMonitor));
    	msg->pMonitor->notify();
    	msg->bMonitorFin = true;
	}
	else
	{
    	msg->sched->put(msg->iCoroId);
	}
 
	return ;
}
```

至此，对应同步调用的网络线程接收响应的工作就结束了，网络线程会回到 CommunicatorEpoll::run () 中，继续等待数据的收发。 综上，客户端同步调用的过程如下图所示。

![图（1-16）同步调用图示](E:\MarkDown\tars框架\picture\up-87adf2eeb121620ed9081a69048ff8042bf.png)

## 异步调用

在 Tars 中，除了最常见的同步调用之外，还可以进行异步调用，异步调用可分三种：普通的异步调用，promise 异步调用与协程异步调用，这里简单介绍普通的异步调用，看看其与上文介绍的同步调用有何异同。

异步调用不会阻塞整个客户端程序，调用完成（请求发送）之后，用户可以继续处理其他事情，等接收到响应之后，Tars 会在异步处理线程当中执行用户实现好的回调函数。在这里，会用到《Effective C++》中条款 35 所介绍的 “藉由 Non-Virtual Interface 手法实现 Template Method 模式”，用户需要继承一个 XXXServantPrxCallback 基类，并实现里面的虚函数，异步回调线程会在收到响应包之后回调这些虚函数，具体的异步调用客户端示例这里不作详细介绍，在 Tars 的 Example 中会找到相应的示例代码。

### 初始化

本文第一章已经详细介绍了客户端的初始化，这里再简单提一下，在第一章的 “1.2.2 初始化代码跟踪 - 2. 执行 Communicator 的初始化函数” 中，已经提到说，在每一个网络线程 CommunicatorEpoll 的初始化过程中，会创建_asyncThreadNum 条异步线程，等待异步调用的时候处理响应数据：

```cpp
CommunicatorEpoll::CommunicatorEpoll(Communicator * pCommunicator,size_t netThreadSeq)
｛
 ......
   //异步线程数
	_asyncThreadNum = TC_Common::strto<size_t>(pCommunicator->getProperty("asyncthread", "3"));
 
	if(_asyncThreadNum == 0)
	{
    	_asyncThreadNum = 3;
	}
 
	if(_asyncThreadNum > MAX_CLIENT_ASYNCTHREAD_NUM)
	{
    	_asyncThreadNum = MAX_CLIENT_ASYNCTHREAD_NUM;
	}
 ......
	//异步队列的大小
	size_t iAsyncQueueCap = TC_Common::strto<size_t>(pCommunicator->getProperty("asyncqueuecap", "10000"));
	if(iAsyncQueueCap < 10000)
	{
    	iAsyncQueueCap = 10000;
	}
 ......
	//创建异步线程
	for(size_t i = 0; i < _asyncThreadNum; ++i)
	{
    	_asyncThread[i] = new AsyncProcThread(iAsyncQueueCap);
    	_asyncThread[i]->start();
	}
 ......
｝
```

在开始讲述异步调用与接收响应之前，先看看大致的调用过程，与图（1-16）的同步调用来个对比。

![图（1-17）客户端同步异步调用图示](E:\MarkDown\tars框架\picture\up-3a9411e3aada449a30524d3060c5b48b6bb.png)

跟同步调用的示例一样，现在有一 MyDemo.StringServer.StringServantObj 的服务，提供一个 RPC 接口是 append，传入两个 string 类型的变量，返回两个 string 类型变量的拼接结果。在执行 tars2cpp 而生成的文件中，定义了回调函数基类 StringServantPrxCallback，用户需要 public 继承这个基类并实现自己的方法，例如：

```cpp
class asyncClientCallback : public StringServantPrxCallback {
public:
  void callback_append(Int32 ret, const string& rStr) {
	cout <<  "append: async callback success and retCode is " << ret << " ,rStr is " << rStr << "\n";
  }
  void callback_append_exception(Int32 ret) {
	cout <<  "append: async callback fail and retCode is " << ret << "\n";
  }
};
```

然后，用户就可以通过 proxy->async_append (new asyncClientCallback (), str1, str2) 进行异步调用了，调用过程与上文的同步调用差不多，函数调用流程如下图所示，可以与图（1-12）进行比较，看看同步调用与异步调用的异同。

![图（1-18）异步调用过程](E:\MarkDown\tars框架\picture\up-333d411aa93bef8f2b5a8060f67d27e87c7.png)

在异步调用中，客户端发起异步调用_proxy->async_append (new asyncClientCallback (), str1, str2) 后，在函数 StringServantProxy::async_append () 中，程序同样会先构造 ServantProxy::tars_invoke_async () 所需要的参数，如请求包类型，RPC 方法名，方法参数等，与同步调用的一个区别是，还传递了承载**回调函数的派生类实例**。接下来便直接调用了 ServantProxy::tars_invoke_async () 方法：

```cpp
tars_invoke_async(tars::TARSNORMAL,"append", _os.getByteBuffer(), context, _mStatus, callback)
```

在 ServantProxy::tars_invoke_async () 方法中，先创建一个 ReqMessage 变量 msg，初始化 msg 变量，给变量赋值，如 Tars 版本号，数据包类型，服务名，RPC 方法名，Tars 的上下文容器，异步调用的超时时间（单位为毫秒）以及异步调用后的回调函数 ServantProxyCallbackPtr callback（等待异步调用返回响应后回调里面的函数）等。最后，与同步调用一样，调用 ServantProxy::invoke () 进行远程方法调用。

在 ServantProxy::invoke () 中，继续填充传递进来的变量 ReqMessage msg。此外，还需要获取调用者 caller 线程的线程私有数据 ServantProxyThreadData，用来指导 RPC 调用。与同步调用一样，利用 ServantProxy::selectNetThreadInfo () 来轮询选取 ObjectProxy（网络线程 CommunicatorEpoll）与对应的 ReqInfoQueue，详细可看同步调用中的介绍，注意区分客户端中的调用者 caller 线程与网络线程，以及之间的通信桥梁。

退出 ServantProxy::selectNetThreadInfo () 后，便得到 ObjectProxy_类型的 pObjProxy 及其对应的 ReqInfoQueue_类型的 ReqInfoQueue，在异步调用中，不需要建立条件变量来阻塞进程，直接通过 pObjectProxy 来发送 RPC 请求，请求信息会暂存在 ReqInfoQueue 中：

```cpp
if(!pReqQ->push_back(msg,bEmpty))
{
	TLOGERROR("[TARS][ServantProxy::invoke msgQueue push_back error num:" << pSptd->_netSeq << "]" << endl);
 
	delete msg;
	msg = NULL;
 
	pObjProxy->getCommunicatorEpoll()->notify(pSptd->_reqQNo, pReqQ);
 
	throw TarsClientQueueException("client queue full");
}
 
pObjProxy->getCommunicatorEpoll()->notify(pSptd->_reqQNo, pReqQ);
```

在之后，就不需要做任何的工作，退出层层函数调用，回到客户端中，程序可以继续执行其他动作。

### 接收响应与函数回调

异步调用的请求发送过程与同步调用的一致，都是在网络线程中通过 ObjectProxy 去调用 AdapterProxy 来发送数据。但是在接收到响应之后，通过图（1-15）可以看到，在函数 AdapterProxy::finishInvoke (ReqMessage*) 中，同步调用会通过 msg->pMonitor->notify () 唤醒客户端的 caller 线程来接收响应包，而在异步调用中，则是如图（1-19）所示，CommunicatorEpoll 与 AsyncProcThread 的关系如图（1-20）所示。

![图（1-19）异步回调的收包处理](E:\MarkDown\tars框架\picture\up-dde2d4dbfe948899ebc771d2e0660aca668.png)

![图（1-20）CommunicatorEpoll与AsyncProcThread](E:\MarkDown\tars框架\picture\up-6ecf09bc767635c96035a415ae8498561fd.png)

在函数 AdapterProxy::finishInvoke (ReqMessage*) 中，程序通过：

```cpp
//异步回调，放入回调处理线程中
_objectProxy->getCommunicatorEpoll()->pushAsyncThreadQueue(msg);
```

将信息包 msg（带响应信息）放到异步回调处理线程中，在 CommunicatorEpoll::pushAsyncThreadQueue () 中，通过轮询的方式选择异步回调处理线程处理接收到的响应包，异步处理线程数默认是 3，最大是 1024。

```cpp
void CommunicatorEpoll::pushAsyncThreadQueue(ReqMessage * msg)
{
	//先不考虑每个线程队列数目不一致的情况
	_asyncThread[_asyncSeq]->push_back(msg);
	_asyncSeq ++;
 
	if(_asyncSeq == _asyncThreadNum)
	{
    	_asyncSeq = 0;
	}
}
```

选取之后，通过 AsyncProcThread::push_back ()，将 msg 包放在响应包队列 AsyncProcThread::_msgQueue 中，然后通过 AsyncProcThread:: notify () 函数通知本异步回调处理线程进行处理，AsyncProcThread:: notify () 函数可以令阻塞在 AsyncProcThread:: run () 中的 AsyncProcThread::timedWait () 的**异步处理线程被唤醒**。

==注意：现在增加了网络线程和异步线程合并的情况，可以直接调用回调，而不需要再notify异步处理线程了==

在 AsyncProcThread::run () 中，主要执行下面的程序进行函数回调：

```cpp
if (_msgQueue->pop_front(msg))
{
 ......
 
	try
	{
    	ReqMessagePtr msgPtr = msg;
    	msg->callback->onDispatch(msgPtr);
	}
	catch (exception& e)
	{
    	TLOGERROR("[TARS][AsyncProcThread exception]:" << e.what() << endl);
	}
	catch (...)
	{
    	TLOGERROR("[TARS][AsyncProcThread exception.]" << endl);
	}
}
```

通过 msg->callback，程序可以调用回调函数基类 StringServantPrxCallback 里面的 onDispatch () 函数。在 StringServantPrxCallback:: onDispatch () 中，分析此次响应所对应的 RPC 方法名，获取响应结果，并通过动态多态，执行用户所定义好的派生类的虚函数。通过 ReqMessagePtr 的引用计数，还可以将 ReqNessage* msg 删除掉，与同步调用不同，同步调用的 msg 的新建与删除都在 caller 线程中，而异步调用的 msg 在 caller 线程上构造，在异步回调处理线程中析构。

\## 总结 TARS 可以在考虑到易用性和高性能的同时快速构建系统并自动生成代码，帮助开发人员和企业以微服务的方式快速构建自己稳定可靠的分布式应用，从而令开发人员只关注业务逻辑，提高运营效率。多语言、敏捷研发、高可用和高效运营的特性使 TARS 成为企业级产品。 《微服务开源框架 TARS 的 RPC 源码解析》系列文章分上下两篇，对 RPC 调用部分进行源码解析。本文是上篇，我们带大家了解了一下 TARS 的客户端。敬请期待下篇《初识 TARS C++ 服务端》

