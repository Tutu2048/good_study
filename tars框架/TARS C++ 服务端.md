# 微服务开源框架 TARS 的 RPC 源码解析 之 初识 TARS C++ 服务端

导语：微服务开源框架 TARS 的 RPC 调用包含客户端与服务端，《微服务开源框架 TARS 的 RPC 源码解析》系列文章将从初识客户端、客户端的同步及异步调用、初识服务端、服务端的工作流程四部分，以 C++ 语言为载体，深入浅出地带你了解 TARS RPC 调用的原理。

## 什么是 TARS

TARS 是腾讯使用十年的微服务开发框架，目前支持 C++、Java、PHP、Node.js、Go 语言。该开源项目为用户提供了涉及到开发、运维、以及测试的一整套微服务平台 PaaS 解决方案，帮助一个产品或者服务快速开发、部署、测试、上线。目前该框架应用在腾讯各大核心业务，基于该框架部署运行的服务节点规模达到数十万。

TARS 的通信模型中包含客户端和服务端。客户端服务端之间主要是利用 RPC 进行通信。本系列文章分上下两篇，对 RPC 调用部分进行源码解析。本文是下篇，我们将以 C++ 语言为载体，带大家了解一下 TARS 的服务端。

## 初识服务端

在使用 TARS 构建 RPC 服务端的时候，TARS 会帮你生成一个 XXXServer 类，这个类是继承自 Application 类的，声明变量 XXXServer g_app，以及调用函数：

```
g_app.main(argc, argv);
g_app.waitForShutdown();
```

便可以开启 TARS 的 RPC 服务了。在开始剖析 TARS 的服务端代码之前，先介绍几个重要的类，让大家有一个大致的认识。

### Application

正如前面所言，一个服务端就是一个 Application，**Application 帮助用户读取配置文件**，根据配置文件初始化代理（假如这个服务端需要调用其他服务，那么就需要初始化代理了）与服务，新建以及启动网络线程与业务线程。

### TC_EpollServer

TC_EpollServer 才是真正的服务端，如果把 Application 比作风扇，那么 TC_EpollServer 就是那个马达。TC_EpollServer 掌管两大模块 —— **网络模块与业务模块**，就是下面即将介绍的两个类。

### NetThread

代表着网络模块，**内含 TC_Epoller 作为 IO 复用**，TC_Socket 建立 socket 连接，ConnectionList 记录众多对客户端的 socket 连接。任何与网络相关的数据收发都与 NetThread 有关。在配置文件中，利用 /tars/application/server 下的 netthread 配置 NetThread 的个数

### HandleGroup 与 Handle

代表着业务模块，**Handle 是执行 PRC 服务的一个线程**，而众多 Handle 组成的 HandleGroup 就是同一个 RPC 服务的一组业务线程了。业务线程负责调用用户定义好的服务代码，并将处理结果放到发送缓存中等待网络模块发送，下文将会详细讲解业务线程如何调用用户定义的代码的，这里用到了简单的 C++ 反射，这点在很多资料中都没有被提及。在配置文件中，利用 /tars/application/server/xxxAdapter 下的 threads 配置一个 HandleGroup 中的 Handle（业务线程）的个数。

### BindAdapter

代表**一个 RPC 服务实体**，在配置文件中的 /tars/application/server 下面的 xxxAdapter 就是对 BindAdapter 的配置，一个 BindAdapter 代表一个服务实体，看其配置就知道 BindAdapter 的作用是什么了，其代表一个 RPC 服务对外的监听套接字，还声明了连接的最大数量，接收队列的大小，业务线程数，RPC 服务名，所使用的协议等。

BindAdapter 本身可以认为是一个服务的实例，能建立真实存在的监听 socket 并对外服务，与网络模块 NetThread 以及业务模块 HandleGroup 都有关联，例如，多个 NetThread 的第一个线程负责对 BindAdapter 的 listen socket 进行监听，有客户连接到 BindAdapter 的 listen socket 就随机在多个 NetThread 中选取一个，将连接放进被选中的 NetThread 的 ConnectionList 中。BindAdapter 则通常会与一组 HandleGroup 进行关联，该 HandleGroup 里面的业务线程就执行 BindAdapter 对应的服务。可见，BindAdapter 与网络模块以及业务模块都有所关联。

好了，介绍完这几个类之后，通过类图看看他们之间的关系：

![图（2-1）服务端相关类图](.\picture\up-5810325ba4ae657aa518ece0566912472e7.png)

服务端 TC_EpollServer 管理类图中左侧的网络模块与右侧的业务模块，前者负责建立与管理服务端的网络关系，后者负责执行服务端的业务代码，两者通过 BindAdapter 构成一个整体，对外进行 RPC 服务。

## 初始化

与客户端一样，服务端也需要进行初始化，来构建上面所说的整体，按照上面的介绍，可以将初始化分为两模块 —— 网络模块的初始化与业务模块的初始化。初始化的所有代码在 Application 的 void main () 以及 void waitForQuit () 中，初始化包括屏蔽 pipe 信号，读取配置文件等，这些将忽略不讲，主要看看其如何通过 epoll 与建立 listen socket 来构建网络部分，以及如何设置业务线程组构建业务部分。

### TC_EpollServer 的初始化

在初始化网络模块与业务模块之前，TC_EpollServer 需要先初始化，主要代码在：

```cpp
void Application::main(int argc, char *argv[])
{
	......
        //初始化Server部分
        initializeServer();
	......
}
```

在 initializeServer () 中会填充 ServerConfig 里面的各个静态成员变量，留待需要的时候取用。可以看到有_epollServer = new TC_EpollServer (iNetThreadNum)，服务端 TC_EpollServer 被创建出来，而且网络线程 NetThread 也被建立出来了：

```cpp
TC_EpollServer::TC_EpollServer(unsigned int iNetThreadNum)
{
    if(_netThreadNum < 1)
    {
        _netThreadNum = 1;
    }
    //网络线程的配置数目不能15个
    if(_netThreadNum > 15)
    {
        _netThreadNum = 15;
    }
 
    for (size_t i = 0; i < _netThreadNum; ++i)
    {
        TC_EpollServer::NetThread* netThreads = new TC_EpollServer::NetThread(this);
        _netThreads.push_back(netThreads);
    }
}
```

此后，其实有一个 AdminAdapter 被建立，但其与一般的 RPC 服务 BindAdapter 不同，这里不展开介绍。

好了，TC_EpollServer 被构建之后，如何给他安排左（网络模块）右（业务模块）护法呢？

### 网络模块初始化

在讲解网络模块之前，再认真地看看网络模块的相关类图：

![图（2-2）网络模块类图](.\picture\up-7b132d464d5f5fbec7db0b45ea371df4c92.png)

先看看 Application 中哪些代码与网络模块的初始化有关吧：

```cpp
void Application::main(int argc, char *argv[])
{
	......
        vector<TC_EpollServer::BindAdapterPtr> adapters;
        //绑定对象和端口
        bindAdapter(adapters);
	......
        _epollServer->createEpoll();
}
 
void Application::waitForShutdown()
{
    waitForQuit();
    ......
}
```

==注意不同👆👇==

- adapters 的创建在initializeAdapter()  -- **createAdapter()**里而且已经赋值了，initializeAdapter还做了bind AdminAdapter
- bindAdapters()

---

网络部分的初始化，离不开建立各 RPC 服务的监听端口（socket，bind，listen），接收客户端的连接（accept），建立 epoll 等。那么何时何地调用这些函数呢？大致过程如下图所示：

![图（2-3）网络模块的初始化](.\picture\up-3f7e87c35829d5537bcbbf565fe55f19208.png)

#### 1. 创建服务实体的 listen socket

首先在 Application::main () 中，调用：

```cpp
vector<TC_EpollServer::BindAdapterPtr> adapters;
//绑定对象和端口
bindAdapter(adapters);
```

在 Application::bindAdapter () 建立一个个服务实体 BindAdapter，通过读取配置文件中的 /tars/application/server 下面的 xxxAdapter 来确定服务实体 BindAdapter 的个数及不同服务实体的配置，然后再调用：

```cpp
BindAdapterPtr bindAdapter = new BindAdapter(_epollServer.get());
_epollServer->bind(bindAdapter);
```

==注意👆 :这里的new已经在createAdapter完成==

来确定服务实体的 listen socket。可以看到，在 TC_EpollServer::bind () 中：

```cpp
int  TC_EpollServer::bind(TC_EpollServer::BindAdapterPtr &lsPtr)
{
    int iRet = 0;
    for(size_t i = 0; i < _netThreads.size(); ++i)
    {
        if(i == 0)
        {
            iRet = _netThreads[i]->bind(lsPtr);
        }
        else
        {
            //当网络线程中listeners没有监听socket时，list使用adapter中设置的最大连接数作为初始化
            _netThreads[i]->setListSize(lsPtr->getMaxConns());
        }
    }
    return iRet;
}
```

==👆注意：这个函数netThread[i]的 NetThread::bind 已经没有了，交由TC_EpollServer::bind  {...BindAdapterPtr的bind}处理==

将上文 TC_EpollServer 的初始化时创建的网络线程组中的第一条网络线程负责创建并监听服务实体的 listen socket，那样就可以避免多线程监听同一个 fd 的惊群效应。

可以看到，接下来继续调用 NetThread::bind (BindAdapterPtr &lsPtr)，其负责做一些准备工作，实际创建 socket 的是在 NetThread::bind (BindAdapterPtr &lsPtr) 中执行的 NetThread::bind (const TC_Endpoint &ep, TC_Socket &s)：

```cpp
void TC_EpollServer::NetThread::bind(const TC_Endpoint &ep, TC_Socket &s)
{
    int type = ep.isUnixLocal()?AF_LOCAL:AF_INET;
 
    if(ep.isTcp())
    {
        s.createSocket(SOCK_STREAM, type);
    }
    else
    {
        s.createSocket(SOCK_DGRAM, type);
    }
 
    if(ep.isUnixLocal())
    {
        s.bind(ep.getHost().c_str());
    }
    else
    {
        s.bind(ep.getHost(), ep.getPort());
    }
 
    if(ep.isTcp() && !ep.isUnixLocal())
    {
        s.listen(1024);
        s.setKeepAlive();
        s.setTcpNoDelay();
        //不要设置close wait否则http服务回包主动关闭连接会有问题
        s.setNoCloseWait();
    }
    s.setblock(false);
}
```

执行到这里，已经创建了服务实体 BindAdapter 的 **listen socket** 了，代码退回到 NetThread::bind (BindAdapterPtr &lsPtr) 后，还可以看到 NetThread 记录 fd 其所负责监听的 BindAdapter：

```cpp
_listeners[s.getfd()] = lsPtr;
```

下图是对创建服务实体的 listen socket 的流程总结

![图（2-4）创建服务实体的listen socket](.\picture\up-3ba8d62ab789a8f18f397ebd084315215c8.png)

#### 2. 创建 epoll

代码回到 Application::main () 中，通过执行：

```cpp
_epollServer->createEpoll();
```

来让 TC_EpollServer 在其掌管的网络线程中建立 epoll：

```cpp
void TC_EpollServer::createEpoll()
{
    for(size_t i = 0; i < _netThreads.size(); ++i)
    {
        _netThreads[i]->createEpoll(i+1);
    }
    //必须先等所有网络线程调用createEpoll()，初始化list后，才能调用initUdp()
    for(size_t i = 0; i < _netThreads.size(); ++i)
    {
        _netThreads[i]->initUdp();
    }
}
```

==createEpoll在waitForShutdown，由服务main调用，但是只看见了初始化连接列表==

代码来到 NetThread::createEpoll (uint32_t iIndex)，这个函数可以作为网络线程 NetThread 的初始化函数，在函数里面建立了网络线程的内存池，创建了 epoll，还将上面创建的 listen socket 加入 epoll 中，当然只有第一条网络线程才有 listen socket，此外还初始化了连接管理链表 ConnectionList _list。看下图对本流程的总结：

![图（2-5）创建epoll](.\picture\up-163fb603bffe3d491f3013c51cc63c364da.png)

#### 3. 启动网络线程

由于 NetThread 是线程，需要执行其 start () 函数才能启动线程。而这个工作不是在 Application::main () 中完成，而是在 Application::waitForShutdown () 中的~~Application::waitForQuit ()~~  startHandle()完成，跟着下面的流程图看代码，就清楚明白了：

![图（2-6）启动网络线程](.\picture\up-2453dc262b02eb44b8b29a05e704919ea48.png)

### 业务模块的初始化

==注意：HandleGroup这个类一句被删除，由BindAdapt 中的vector<HandlePtr > _handles来管理，Handle类也剔除了HandleGroupPtr这个变量==

同样，与网络模块一样，在讲解业务模块之前，先认真地看看业务模块的相关类图：

![图（2-7）业务模块相关类图](.\picture\up-368c58fc3c1a7b36ac23ef52a43569adee2.png)

在业务模块初始化中，我们需要理清楚两个问题：**业务模块如何与用户填充实现的 XXXServantImp 建立联系**，从而使请求到来的时候，Handle 能够调用用户定义好的 RPC 方法？**业务线程在何时何地被启动**，如何等待着请求的到达？

看看 Application 中哪些代码与业务模块的初始化有关吧：

```cpp
void Application::main(int argc, char *argv[])
{
	......
 
        vector<TC_EpollServer::BindAdapterPtr> adapters;
        bindAdapter(adapters);
        //业务应用的初始化
        initialize();
 
 
        //设置HandleGroup分组，启动线程
        for (size_t i = 0; i < adapters.size(); ++i)
        {
            string name = adapters[i]->getName();
 
            string groupName = adapters[i]->getHandleGroupName();
 
            if(name != groupName)
            {
                TC_EpollServer::BindAdapterPtr ptr = _epollServer->getBindAdapter(groupName);
 
                if (!ptr)
                {
                    throw runtime_error("[TARS][adater `" + name + "` setHandle to group `" + groupName + "` fail!");
                }
 
            }
            setHandle(adapters[i]);
        }
 
        //启动业务处理线程
        _epollServer->startHandle();
	......
 
}
```

在 bindAdapter () 与 initialize ()  解决了前面提到的第一个问题*，剩下的代码实现了 handle 业务线程组的创建与启动。

==注意：Application::initiallizeAdapter中的createAdapter（完成了创建、设置、映射）和waitForShutdown里startHandle（），ServantHelperManager也不在设计为单例，直接聚合在Application里==

#### 1. 将 BindAdapter 与用户定义的方法关联起来

如何进行关联？先看看下面的代码流程图：

![图（2-8）通过ServantHelperManager关联BindAdapter与服务Servant](.\picture\up-a1b0c484583b4187417e762f3800428da93.png)

如何让业务线程能够调用用户自定义的代码？这里引入了 ServantHelperManager，先简单剧透一下，通过 ServantHelperManager 作为桥梁，业务线程可以通过 BindAdapter 的 ID 索引到服务 ID，然后通过服务 ID 索引到用户自定义的 XXXServantImp 类的生成器，有了生成器，业务线程就可以生成 XXXServantImp 类并调用里面的方法了。下面一步一步分析。

在 Application::main () 调用的 Application::bindAdapter () 中看到有下面的代码：

```cpp
for (size_t i = 0; i < adapterName.size(); i++)
{
	……
     string servant = _conf.get("/tars/application/server/" + adapterName[i] + "<servant>");
     checkServantNameValid(servant, sPrefix);
 
     ServantHelperManager::getInstance()->setAdapterServant(adapterName[i], servant);
	……
}
```

举个例子，adapterNamei 为 **MyDemo.StringServer.StringServantAdapter**，而 servant 为 **MyDemo.StringServer.StringServantObj**，这些都是在配置文件中读取的，前者是 BindAdapter 的 ID，而后者是服务 ID。在 ServantHelperManager:: setAdapterServant () 中，仅仅是执行：

```cpp
void ServantHelperManager::setAdapterServant(const string &sAdapter, const string &sServant)
{
    _adapter_servant[sAdapter] = sServant;
    _servant_adapter[sServant] = sAdapter;
}
```

而这两个成员变量仅仅是：

```cpp
    /**
     * Adapter包含的Servant(Adapter名称:servant名称)
     */
    map<string, string>                     _adapter_servant;
 
    /**
     * Adapter包含的Servant(Servant名称:Adapter名称)
     */
map<string, string>                     _servant_adapter;
```

在这里仅仅是作一个映射记录，后续可以通过 BindAdapter 的 ID 可以索引到服务的 ID，通过服务的 ID 可以利用简单的 C++ 反射得出用户实现的 XXXServantImp 类，从而得到用户实现的方法。

如何实现从服务 ID 到类的反射？同样需要通过 ServantHelperManager 的帮助。在 Application::main () 中，执行完 Application::bindAdapter () 会执行 initialize ()，这是一个纯虚函数，实际会执行派生类 XXXServer 的函数，类似：

```cpp
void
StringServer::initialize()
{
    //initialize application here:
    //...
    addServant<StringServantImp>(ServerConfig::Application + "." + ServerConfig::ServerName + ".StringServantObj");
}
```

代码最终会执行 ServantHelperManager:: addServant<T>()：

```cpp
template<typename T>
    void addServant(const string &id,bool check = false)
    {
        if(check && _servant_adapter.end() == _servant_adapter.find(id))
        {
            cerr<<"[TARS]ServantHelperManager::addServant "<< id <<" not find adapter.(maybe not conf in the web)"<<endl;
            throw runtime_error("[TARS]ServantHelperManager::addServant " + id + " not find adapter.(maybe not conf in the web)");
        }
        _servant_creator[id] = new ServantCreation<T>();
}
```

其中参数 const string& id 是服务 ID，例如上文的 **MyDemo.StringServer.StringServantObj**，T 是用户填充实现的 XXXServantImp 类。

上面代码的_servant_creatorid = new ServantCreation<T>() 是函数的关键，_servant_creator 是 map<string, ServantHelperCreationPtr>，可以通过服务 ID 索引到 ServantHelperCreationPtr，而 ServantHelperCreationPtr 是什么？是帮助我们生成 XXXServantImp 实例的类生成器，这就是简单的 C++ 反射：

```cpp
/**
 * Servant
 */
class ServantHelperCreation : public TC_HandleBase
{
public:
    virtual ServantPtr create(const string &s) = 0;
};
typedef TC_AutoPtr<ServantHelperCreation> ServantHelperCreationPtr;
 
//////////////////////////////////////////////////////////////////////////////
/**
 * Servant
 */
template<class T>
struct ServantCreation : public ServantHelperCreation
{
    ServantPtr create(const string &s) { T *p = new T; p->setName(s); return p; }
};
```

以上就是通过服务 ID 生成相应 XXXServantImp 类的简单反射技术，业务线程组里面的业务线程只需要获取到所需执行的业务的 BindAdapter 的 ID，就可以通过 ServantHelperManager 获得服务 ID，有了服务 ID 就可以获取 XXXServantImp 类的生成器从而生成 XXXServantImp 类执行里面由用户定义好的 RPC 方法。现在重新看图（2-8）就大致清楚整个流程了。

#### 2.Handle 业务线程的启动

剩下的部分就是 HandleGroup 的创建，并将其与 BindAdapter 进行相互绑定关联，同时也需要绑定到 TC_EpollServer 中，随后创建 / 启动 HandleGroup 下面的 Handle 业务线程，启动 Handle 的过程涉及上文 “将 BindAdapter 与用户定义的方法关联起来” 提到的获取服务类生成器。先看看大致的代码流程图：

![img](.\picture\s6xc0y60a4-17271614137091.png)

在这里分两部分，第一部分是在 Application::main () 中执行下列代码：

```cpp
//设置HandleGroup分组，启动线程
for (size_t i = 0; i < adapters.size(); ++i)
{
    string name = adapters[i]->getName();
    string groupName = adapters[i]->getHandleGroupName();
 
    if(name != groupName)
    {
        TC_EpollServer::BindAdapterPtr ptr = _epollServer->getBindAdapter(groupName);
 
        if (!ptr)
        {
            throw runtime_error("[TARS][adater `" + name + "` setHandle to group `" + groupName + "` fail!");
        }
 
    }
    setHandle(adapters[i]);
}
```

遍历在配置文件中定义好的每一个 BindAdapter（例如 **MyDemo.StringServer.StringServantAdapter**），并为其设置业务线程组 HandleGroup，让线程组的所有线程都可以执行该 BindAdapter 所对应的 RPC 方法。跟踪代码如下：

```cpp
void Application::setHandle(TC_EpollServer::BindAdapterPtr& adapter)
{
    adapter->setHandle<ServantHandle>();
}
```

注意，ServantHandle 是 Handle 的派生类，就是业务处理线程类，随后来到：

```cpp
template<typename T> void setHandle()
{
    _pEpollServer->setHandleGroup<T>(_handleGroupName, _iHandleNum, this);
}
```

真正创建业务线程组 HandleGroup 以及组内的线程，并将线程组与 BindAdapter，TC_EpollServer 关联起来的代码在 TC_EpollServer:: setHandleGroup () 中：

```cpp
/**
 * 创建一个handle对象组，如果已经存在则直接返回
 * @param name
 * @return HandlePtr
 */
template<class T> void setHandleGroup(const string& groupName, int32_t handleNum, BindAdapterPtr adapter)
{
    map<string, HandleGroupPtr>::iterator it = _handleGroups.find(groupName);
 
    if (it == _handleGroups.end())
    {
        HandleGroupPtr hg = new HandleGroup();
        hg->name = groupName;
        adapter->_handleGroup = hg;
 
        for (int32_t i = 0; i < handleNum; ++i)
        {
            HandlePtr handle = new T();
            handle->setEpollServer(this);
            handle->setHandleGroup(hg);
            hg->handles.push_back(handle);
        }
 
        _handleGroups[groupName] = hg;
        it = _handleGroups.find(groupName);
    }
    it->second->adapters[adapter->getName()] = adapter;
    adapter->_handleGroup = it->second;
}
```

在这里，可以看到业务线程组的创建：HandleGroupPtr hg = new HandleGroup ()；业务线程的创建：HandlePtr handle = new T ()（T 是 ServantHandle）；建立关系，例如 BindAdapter 与 HandleGroup 的相互关联：it->second->adaptersadapter->getName () = adapter 和 adapter->_handleGroup = it->second。执行完上面的代码，就可以得到下面的类图了：

![图（2-10）再看业务模块相关类图](.\picture\up-e51a0947aedb952dde422dfbe1d59271e5b.png)

这里再通过函数流程图简单复习一下上述代码的流程，主要内容均在 TC_EpollServer:: setHandleGroup () 中：

![图（2-11）建立业务模块](.\picture\up-99100e36887c288a8fc2d49fa43e32ae5f6.png)

随着函数的层层退出，代码重新来到 Application::main () 中，随后执行：

```cpp
//启动业务处理线程
_epollServer->startHandle();
```

在 TC_EpollServer::startHandle () 中，遍历 TC_EpollServer 控制的业务模块 HandleGroup 中的所有业务线程组，并遍历组内的各个 Handle，执行其 start () 方法进行线程的启动：

```cpp
void TC_EpollServer::startHandle()
{
    if (!_handleStarted)
    {
        _handleStarted = true;
 
        for (auto& kv : _handleGroups)
        {
            auto& hds = kv.second->handles;
            for (auto& handle : hds)
            {
                if (!handle->isAlive())
                    handle->start();
            }
        }
    }
}
```

由于 Handle 是继承自 TC_Thread 的，在执行 Handle::start () 中，会执行虚函数 ~~Handle::run ()~~，在 Handle::run () 中主要是执行两个函数，一个是 ServantHandle::initialize ()，另一个是 Handle::handleImp ()：

```cpp
void TC_EpollServer::Handle::run()
{
    initialize();
 
    handleImp();
}
```

ServantHandle::initialize () 的主要作用是取得用户实现的 RPC 方法，其实现原理与上文（“2.2.3 业务模块的初始化” 中的第 1 小点 “将 BindAdapter 与用户定义的方法关联起来”）提及的一样，借助与其关联的 BindAdapter 的 ID 号，以及 ServantHelpManager，来查找到用户填充实现的 XXXServantImp 类的生成器并生成 XXXServantImp 类的实例，将这个实例与服务名构成 pair <string, ServantPtr > 变量，放进 map<string, ServantPtr> ServantHandle:: _servants 中，等待业务线程 Handle 需要执行用户自定义方法的时候，从 map<string, ServantPtr> ServantHandle:: _servants 中查找：

```cpp
void ServantHandle::initialize()
{
    map<string, TC_EpollServer::BindAdapterPtr>::iterator adpit;
    // 获取本Handle所关联的BindAdapter
    map<string, TC_EpollServer::BindAdapterPtr>& adapters = _handleGroup->adapters;
    // 遍历所有BindAdapter
    for (adpit = adapters.begin(); adpit != adapters.end(); ++adpit)
    {
        // 借助ServantHelperManager来获取服务指针——XXXServantImp类的指针
        ServantPtr servant = ServantHelperManager::getInstance()->create(adpit->first);
        // 将指针放进map<string, ServantPtr> ServantHandle:: _servants中
        if (servant)
        {
            _servants[servant->getName()] = servant;
        }
        else
        {
            TLOGERROR("[TARS]ServantHandle initialize createServant ret null, for adapter `" + adpit->first + "`" << endl);
        }
    }
 
    ......
}
```

而 Handle::handleImp () 的主要作用是使业务线程阻塞在等待在条件变量上，在这里，可以看到_handleGroup->monitor.timedWait (_iWaitTime) 函数，阻塞等待在条件变量上：

```cpp
void TC_EpollServer::Handle::handleImp()
{
    ......
    struct timespec ts;
 
    while (!getEpollServer()->isTerminate())
    {
        {
            TC_ThreadLock::Lock lock(_handleGroup->monitor);
 
            if (allAdapterIsEmpty() && allFilterIsEmpty())
            {
                _handleGroup->monitor.timedWait(_iWaitTime);
            }
        }
}
......
}
```

Handle 线程通过条件变量来让所有业务线程阻塞等待被唤醒 ，因为本章是介绍初始化，因此代码解读到这里先告一段落，稍后再详解服务端中的业务线程 Handle 被唤醒后，如何通过 map<string, ServantPtr> ServantHandle:: _servants 查找并执行业务。现在通过函数流程图复习一下上述的代码流程：

![图（2-12）启动Handle业务线程](.\picture\up-ce126d5eaa96141f6b392b2f338e34ebf68.png)

==tc_transceiver.h No.50行的注释，对于下面的内容很重要==

## 服务端的工作

经过了初始化工作后，服务端就进入工作状态了，服务端的工作线程分为两类，正如前面所介绍的网络线程与业务线程，网络线程负责接受客户端的连接与收发数据，而业务线程则只关注执行用户所定义的 PRC 方法，两种线程在初始化的时候都已经执行 start () 启动了。

大部分服务器都是按照 accept ()->read ()->write ()->close () 的流程执行的，大致工作流程图如下图所示：

![图（2-13）普通服务器工作流程](.\picture\up-ec97fba53a135cc25d857940b814573b324.png)

TARS 的服务端也不例外。

判定逻辑采用 Epoll IO 复用模型实现，**每一条网络线程 NetThread 都有一个 TC_Epoller 来做事件的收集、侦听、分发**。

正如前面所介绍，只有第一条网络线程会执行连接的监听工作，接受新的连接之后，就会构造一个 Connection 实例，并选择处理这个连接的网络线程。

请求被读入后，将暂存在接收队列中，并通知业务线程进行处理，在这里，业务线程终于登场了，处理完请求后，将结果放到发送队列。

发送队列有数据，自然需要通知网络线程进行发送，接收到发送通知的网络线程会将响应发往客户端。

TARS 服务器的工作流程大致就是如此，如上图所示的普通服务器工作流程没有多大的区别，下面将按着接受客户端连接，读入 RPC 请求，处理 RPC 请求，发送 RPC 响应四部分逐一介绍介绍服务端的工作。

### 接受客户端连接

讨论服务器接受请求，很明显是从网络线程（而且是网络线程组的第一条网络线程）的 NetThread::run () 开始分析，在上面说到的创建 TC_Epoller 并将监听 fd 放进 TC_Epoller 的时候，执行的是：

```cpp
_epoller.add(kv.first, H64(ET_LISTEN) | kv.first, EPOLLIN);
```

那么从 epoll_wait () 返回的时候，epoll_event 中的联合体 epoll_data 将会是 (ET_LISTEN | listen socket’fd)，从中获取高 32 位，就是 ET_LISTEN，然后执行下面 switch 中 case ET_LISTEN 的分支

```cpp
try
{
    const epoll_event &ev = _epoller.get(i);
    uint32_t h = ev.data.u64 >> 32;
 
    switch(h)
    {
    case ET_LISTEN:
        {
            //监听端口有请求
            auto it = _listeners.find(ev.data.u32);
            if( it != _listeners.end())
            {
                if(ev.events & EPOLLIN)
                {
                    bool ret;
                    do
                    {
                        ret = accept(ev.data.u32);
                    }while(ret);
                }
            }
        }
        break;
    case ET_CLOSE:
        //关闭请求
        break;
    case ET_NOTIFY:
        //发送通知
        ......
        break;
     case ET_NET:
        //网络请求
         ......
        break;
      default:
         assert(true);
      }
}
```

而 ret = accept (ev.data.u32) 的整个函数流程如下图所示（ev.data.u32 就是被激活的 BindAdapter 对应的监听 socket 的 fd）：

![图（2-14）服务端accept一位客户端](.\picture\up-98c7ff66f00da184a54e6362c06d50e4fbc.png)

在讲解之前，先复习一下网络线程相关类图，以及通过图解对 accept 有个大致的印象：

![图（2-15）网络模块类图](.\picture\up-56e98eed680e8a988d3dfec0ed4ddcbcf86.png)

![图（2-16）服务端接受一个客户端连接](.\picture\up-619535a64353138a71fc87a20a7b3afaa78.png)

好了，跟着图（2-14），现在从 NetThread::run () 的 NetThread::accept (int fd) 讲起。

==上面到此，改动较大==

#### 1.accept 获取客户端 socket

进入 NetThread::accept (int fd)，可以看到代码执行了：

```cpp
//接收连接
TC_Socket s;
s.init(fd, false, AF_INET);
int iRetCode = s.accept(cs, (struct sockaddr *) &stSockAddr, iSockAddrSize);
```

通过 TC_Socket::accept ()，调用系统函数 accept () 接受了客户端的辛辛苦苦三次握手来的 socket 连接，然后对客户端的 IP 与端口进行打印以及检查，并分析对应的 BindAdapter 是否过载，过载则关闭连接。随后对客户端 socket 进行设置：

```cpp
cs.setblock(false);
cs.setKeepAlive();
cs.setTcpNoDelay();
cs.setCloseWaitDefault();
```

到此，对应图（2-16）的第一步 —— 接受客户端连接（流程如下图所示），已经完成。

![图（2-17）accept客户端](.\picture\up-e95c4cb8fcdd7548af2e277b8f92f301d8e.png)

#### 2. 为客户端 socket 创建 Connection

接下来是为新来的客户端 socket 创建一个 Connection，在 NetThread::accept (int fd) 中，创建 Connection 的代码如下：

```cpp
int timeout = _listeners[fd]->getEndpoint().getTimeout()/1000;
 
Connection *cPtr = new Connection(_listeners[fd].get(), fd, (timeout < 2 ? 2 : timeout), cs.getfd(), ip, port);
```

构造函数中的参数依次是，这次新客户端所对应的 BindAdapter 指针，BindAdapter 对应的 listen socket 的 fd，超时时间，客户端 socket 的 fd，客户端的 ip 以及端口。在 Connection 的构造函数中，通过 fd 也关联其 TC_Socket：

```cpp
// 服务连接
Connection::Connection(TC_EpollServer::BindAdapter *pBindAdapter, int lfd, int timeout, int fd, const string& ip, uint16_t port)
{
    ......
    _sock.init(fd, true,90 AF_INET);
}
```

那么关联 TC_Socket 之后，通过 Connection 实例就可以操作的客户端 socket 了。至此，对应图（2-16）的第二步 —— 为客户端 socket 创建 Connection 就完成了（流程如下图所示）。

![图（2-18）创建Connection](.\picture\up-8e882c15e88fc8a7d8f0c29e79efd8c1d2b.png)

#### 3. 为 Connection 选择一条网络线程

最后，就是为这个 Connection 选择一个网络线程，将其加入网络线程对应的 ConnectionList，在 NetThread::accept (int fd) 中，执行：

```cpp
//addTcpConnection(cPtr);
_epollServer->addConnection(cPtr, cs.getfd(), TCP_CONNECTION);
```

==已经去除`addConnection`该层函数，直接调用`addTcpConnection`、`addUdpConnection`==
~~TC_EpollServer::addConnection ()~~ 的代码如下所示：

```cpp
void TC_EpollServer::addConnection(TC_EpollServer::NetThread::Connection * cPtr, int fd, int iType)
{
    TC_EpollServer::NetThread* netThread = getNetThreadOfFd(fd);
 
    if(iType == 0)
    {
        netThread->addTcpConnection(cPtr);
    }
    else
    {
        netThread->addUdpConnection(cPtr);
    }
}
```

看到，先为 Connection* cPtr 选择网络线程，在流程图中，被选中的网络线程称为 Chosen_NetThread。选网络线程的函数是 TC_EpollServer::getNetThreadOfFd (int fd)，根据客户端 socket 的 fd 求余数得到，具体代码如下：

```cpp
NetThread* getNetThreadOfFd(int fd)
{
    return _netThreads[fd % _netThreads.size()];
}
```cpp

接着调用被选中线程的 NetThread::addTcpConnection () 方法（或者

NetThread::addUdpConnection ()，这里只介绍 TCP 的方法），将 Connection 加入被选中网络线程的 ConnectionList 中，最后会执行_epoller.add (cPtr->getfd (), cPtr->getId (), EPOLLIN | EPOLLOUT) 将客户端 socket 的 fd 加入本网络线程的 TC_Epoller 中，让本网络线程负责对本客户端的数据收发。至此对应图（28）的第三步就执行完毕了（具体流程如下图所示）。

![图（2-19）为Connection选择一个网络线程](.\picture\up-4942287cb3a62714e37829089fb77faf9f7.png)

### 接收 RPC 请求

讨论服务器接收 RPC 请求，同样从网络线程的 NetThread::run () 开始分析，上面是进入 switch 中的 case ET_LISTEN 分支来接受客户端的连接，那么现在就是进入 case ET_NET 分支了，为什么是 case ET_NET 分支呢？因为上面提到，将客户端 socket 的 fd 加入 TC_Epoller 来监听其读写，采用的是_epoller.add (cPtr->getfd (), cPtr->getId (), EPOLLIN | EPOLLOUT)，传递给函数的第二个参数是 32 位的整形 cPtr->getId ()，而函数的第二个参数要求必须是 64 位的整型，因此，这个参数将会是高 32 位是 0，低 32 位是 cPtr->getId () 的 64 位整形。而第二个参数的作用是当该注册的事件引起 epoll_wait () 退出的时候，会作为激活事件 epoll_event 结构体中的 64 位联合体 epoll_data_t data 返回给用户。因此，看下面 NetThread::run () 代码：

```cpp
try
{
    const epoll_event &ev = _epoller.get(i);
    uint32_t h = ev.data.u64 >> 32;
 
    switch(h)
    {
    case ET_LISTEN:
        ……
        break;
    case ET_CLOSE:
        //关闭请求
        break;
    case ET_NOTIFY:
        //发送通知
        ......
        break;
     case ET_NET:
        //网络请求
        processNet(ev);
        break;
      default:
         assert(true);
      }
}
```

代码中的 h 是 64 位联合体 epoll_data_t data 的高 32 位，经过上面分析，客户端 socket 若因为接收到数据而引起 epoll_wait () 退出的话，epoll_data_t data 的高 32 位是 0，低 32 位是 cPtr->getId ()，因此 h 将会是 0。而 ET_NET 就是 0，因此客户端 socket 有数据来到的话，会执行 case ET_NET 分支。下面看看执行 case ET_NET 分支的函数流程图。

![图（2-20）服务端接收RPC请求流程图](.\picture\up-1830e4f320d82ff58c78a30dcff31a50aaa.png)

==注意：TC_TCP[UDP]Transceiver::doResponse->TC_Transceiver::doProtocolAnalysis现在将收发包的逻辑交由TC_Transceiver类来控制，绑定Connection::onParserCallback回调，解析数据后得到完整包的内容，to insertRecvQueue ，然后在去处理_onCompletePackageCallback,，而一开始的doResponse是由ServantHandle类来调用的==

#### 1. 获取激活了的连接 Connection

收到 RPC 请求，进入到 NetThread::processNet ()，服务器需要知道是哪一个客户端 socket 被激活了，因此在 NetThread::processNet () 中执行：

```cpp
void TC_EpollServer::NetThread::processNet(const epoll_event &ev)
{
    uint32_t uid = ev.data.u32;
 
    Connection *cPtr = getConnectionPtr(uid);
    //......
}
```

正如上面说的，epoll_data_t data 的高 32 位是 0，低 32 位是 cPtr->getId ()，那么获取到 uid 之后，通过 NetThread::getConnectionPtr () 就可以从 ConnectionList 中返回此时此刻所需要读取 RPC 请求的 Connection 了。之后对获取的 Connection 进行简单的检查工作，并看看 epoll_event::events 是否是 EPOLLERR 或者 EPOLLHUP（具体流程如下图所示）。

![图（2-21）获取收到数据的Connection](.\picture\up-55f48d0a6c929ddfc67cd49708739c38101.png)

#### 2. 接收客户端请求，放进线程安全队列中

接着，就需要接收客户端的请求数据了，有数据接收意味着 epoll_event::events 是 EPOLLIN，看下面代码，主要是 NetThread::recvBuffer () 读取 RPC 请求数据，以及以及 Connection:: insertRecvQueue () 唤醒业务线程发送数据。

```cpp
if(ev.events & EPOLLIN)               //有数据需要读取
{
    recv_queue::queue_type vRecvData;
    int ret = recvBuffer(cPtr, vRecvData);
 
    if(ret < 0)
    {
        delConnection(cPtr,true,EM_CLIENT_CLOSE);
        return;
    }
 
    if(!vRecvData.empty())
    {
        cPtr->insertRecvQueue(vRecvData);
    }
}
```

先看看 NetThread::recvBuffer ()，首先服务端会先创建一个线程安全队列来承载接收到的数据 recv_queue::queue_type vRecvData，再将刚刚获取的 Connection cPtr 以及 recv_queue::queue_type vRecvData 作为参数调用 NetThread::recvBuffer (cPtr, vRecvData)。

而 NetThread::recvBuffer () 进一步调用 Connection::recv () 函数：

```cpp
int  NetThread::recvBuffer(NetThread::Connection *cPtr, recv_queue::queue_type &v)
{
    return cPtr->recv(v);
}
```

Connection::recv () 会依照不同的传输层协议（若 UDP 传输，lfd==-1），执行不同的接收方法，例如 TCP 会执行：

```cpp
iBytesReceived = ::read(_sock.getfd(), (void*)buffer, sizeof(buffer))
```

根据数据接收情况，如收到 FIN 分节，errno==EAGAIN 等执行不同的动作。若收到真实的请求信息包，会将接收到的数据放在 string Connection::_recvbuffer 中，然后调用 Connection:: parseProtocol ()。

在 Connection:: parseProtocol () 中会回调协议解析函数对接收到的数据进行检验，检验通过后，会构造线程安全队列中的元素 tagRecvData* recv，并将其放进线程安全队列中：

```cpp
tagRecvData* recv = new tagRecvData();
recv->buffer           = std::move(ro);
recv->ip               = _ip;
recv->port             = _port;
recv->recvTimeStamp    = TNOWMS;
recv->uid              = getId();
recv->isOverload       = false;
recv->isClosed         = false;
recv->fd               = getfd();
//收到完整的包才算
this->_bEmptyConn = false;
 
//收到完整包
o.push_back(recv);
```

到此，RPC 请求数据已经被完全获取并放置在线程安全队列中（具体过程如下图所示）。

![图（2-22）接收客户请求](.\picture\up-d55689a3710196d079d731aa1048a99e85c.png)

#### 3. 线程安全队列非空，唤醒业务线程发送

代码运行至此，线程安全队列里面终于有 RPC 请求包数据了，可以唤醒业务线程 Handle 进行处理了，代码回到 NetThread::processNet ()，只要线程安全队列非空，就执行 Connection:: insertRecvQueue ()：

```cpp
void NetThread::processNet(const epoll_event &ev)
{
    ......
 
    if(ev.events & EPOLLIN)               //有数据需要读取
    {
        ......
 
        if(!vRecvData.empty())
        {
            cPtr->insertRecvQueue(vRecvData);
        }
    }
    ......
}
```

在 Connection:: insertRecvQueue () 中，会先对 BindAdapter 进行过载判断，分为未过载，半过载以及全过载三种情况。若全过载会丢弃线程安全队列中的所有 RPC 请求数据，否则会执行 BindAdapter::insertRecvQueue ()。

在 BindAdapter::insertRecvQueue () 中，代码主要有两个动作，第一个是将获取到的 RPC 请求包放进 BindAdapter 的接收队列 ——recv_queue _rbuffer 中：

```cpp
_rbuffer.push_back(vtRecvData)
```

第二个是唤醒等待条件变量的 HandleGroup 线程组：

```cpp
_handleGroup->monitor.notify()
```

现在，服务端的网络线程在接收 RPC 请求数据后，终于唤醒了业务线程（具体流程看下图所示），接下来轮到业务模块登场，看看如何处理 RPC 请求了。

![图（2-23）唤醒业务线程](.\picture\up-99e791cf8ad3899b16a2efd6c38fcdbc8e3.png)

### 处理 RPC 请求

与前文接收到请求数据后，唤醒业务线程组 HandleGroup（就是刚刚才介绍完的_handleGroup->monitor.notify ()）遥相呼应的地方是在 “2.2.3 业务模块的初始化” 第 2 小点 “Handle 业务线程的启动” 中提到的，在 Handle::handleImp () 函数中的_handleGroup->monitor.timedWait (_iWaitTime)。通过条件变量，业务线程组 HandleGroup 里面的业务线程一起阻塞等待着网络线程对其发起唤醒。现在，终于对条件变量发起通知了，接下来将会如何处理请求呢？在这里，需要先对 2.2.3 节进行复习，了解到 ServantHandle::_servants 里面究竟承载着什么。

好了，处理 RPC 请求分为三步：构造请求上下文，调用用户实现的方法处理请求，将响应数据包 push 到线程安全队列中并通知网络线程，具体函数流程如下图所示，现在进一步分析：

![图（2-24）服务端处理RPC请求流程图](.\picture\up-8745cc23736828b3b58ff1464de49f526ae.png)

#### 1. 获取请求数据构造请求上下文

当业务线程从条件变量上被唤醒之后，从其负责的 BindAdapter 中获取请求数据：adapter->waitForRecvQueue (recv, 0)，在 BindAdapter::waitForRecvQueue () 中，将从线程安全队列 recv_queue BindAdapter::_ rbuffer 中获取数据：

```
bool BindAdapter::waitForRecvQueue(tagRecvData* &recv, uint32_t iWaitTime)
{
    bool bRet = false;
 
    bRet = _rbuffer.pop_front(recv, iWaitTime);
 
    if(!bRet)
    {
        return bRet;
    }
 
    return bRet;
}
```

还记得在哪里将数据压入线程安全队列的吗？对，就在 “2.3.2 接收 RPC 请求” 的第 3 点 “线程安全队列非空，唤醒业务线程发送” 中。

接着，调用 ServantHandle::handle () 对接收到的 RPC 请求数据进行处理。

处理的第一步正如本节小标题所示 —— 构造请求上下文，用的是 ServantHandle::createCurrent ()：

```
void ServantHandle::handle(const TC_EpollServer::tagRecvData &stRecvData)
{
    TarsCurrentPtr current = createCurrent(stRecvData);
 
......
}
```

在 ServantHandle::createCurrent () 中，先 new 出 TarsCurrent 实例，然后调用其 initialize () 方法，在 TarsCurrent::initialize (const TC_EpollServer::tagRecvData &stRecvData, int64_t beginTime) 中，将 RPC 请求包的内容放进请求上下文 TarsCurrentPtr current 中，后续只需关注这个请求上下文即可。另外可以稍微关注一下，若采用 TARS 协议会使用 TarsCurrent::initialize (const string &sRecvBuffer) 将请求包的内容放进请求上下文中，否则直接采用 memcpy () 系统调用来拷贝内容。下面稍微总结一下这小节的流程：

![图（2-25）构造请求上下文](.\picture\up-3cf6f254b92a178f9d68d31597bc6cfd059.png)

#### 2. 处理请求（只介绍 TARS 协议）

当获取到请求上下文之后，就需要对其进行处理了。

```
void ServantHandle::handle(const TC_EpollServer::tagRecvData &stRecvData)
{
    // 构造请求上下文
    TarsCurrentPtr current = createCurrent(stRecvData);
 
    if (!current) return;
    // 处理请求
    if (current->getBindAdapter()->isTarsProtocol())
    {
        handleTarsProtocol(current);
    }
    else
    {
        handleNoTarsProtocol(current);
    }
}
```

本 RPC 框架支持 TARS 协议与非 TARS 协议，下面只会介绍对 TARS 协议的处理，对于非 TARS 协议，分析流程也是差不多，对非 TARS 协议协议感兴趣的读者可以对比着来分析非 TARS 协议部分。在介绍之前，先看看服务相关的继承体系，下面不要混淆这三个类了：

![图（2-26）服务类继承体系](.\picture\up-98ca0f057d5b0eb6e38218efd079dc27b0e.png)

好了，现在重点放在 ServantHandle::handleTarsProtocol (const TarsCurrentPtr ¤t) 函数上面。先贴代码：

```
void ServantHandle::handleTarsProtocol(const TarsCurrentPtr &current)
{
    // 1-对请求上下文current进行预处理
    // 2-寻找合适的服务servant
    map<string, ServantPtr>::iterator sit = _servants.find(current->getServantName());
 
    if (sit == _servants.end())
    {
        current->sendResponse(TARSSERVERNOSERVANTERR);
 
        return;
    }
 
    int ret = TARSSERVERUNKNOWNERR;
 
    string sResultDesc = "";
 
    vector<char> buffer;
 
    try
    {
        //3-业务逻辑处理
        ret = sit->second->dispatch(current, buffer);
    }
    catch(TarsDecodeException &ex)
    {
       ……
    }
    catch(TarsEncodeException &ex)
    {
        ……
    }
    catch(exception &ex)
    {
       ……
    }
    catch(...)
    {
        ……
    }
//回送响应，第3小点再分析吧
……
}
```

进入函数中，会先对请求上下文进行预处理，例如 set 调用合法性检查，染色处理等。随后，就依据上下文中的服务名来获取服务对象：map<string, ServantPtr>::iterator sit = _servants.find (current->getServantName ())，_servants 在 “2.2.3 业务模块的初始化” 第 2 小点 “Handle 业务线程的启动” 中被赋予内容，其 key 是服务 ID（或者叫服务名），value 是用户实现的服务 XXXServantImp 实例指针。

随后就可以利用 XXXServantImp 实例指针来执行 RPC 请求了：ret = sit->second->dispatch (current, buffer)，在 Servant:: dispatch ()（如图（2-26）因为 XXXServantImp 是继承自 XXXServant，而 XXXServant 继承自 Servant，所以实际是执行 Servant 的方法）中，使用不同的协议会有不同的处理方式，这里只介绍 TARS 协议的，调用了 XXXServant::onDispatch (tars::TarsCurrentPtr _current, vector<char> &_sResponseBuffer) 方法：

```
int Servant::dispatch(TarsCurrentPtr current, vector<char> &buffer)
{
    int ret = TARSSERVERUNKNOWNERR;
 
    if (current->getFuncName() == "tars_ping")
    {
        //略
    }
    else if (!current->getBindAdapter()->isTarsProtocol())
    {
        //略
    }
    else
    {
        TC_LockT<TC_ThreadRecMutex> lock(*this);
 
        ret = onDispatch(current, buffer);
    }
    return ret;
}
```

XXXServant 类就是执行 Tars2Cpp 的时候生成的，会依据用户定义的 tars 文件来生成相应的纯虚函数，以及 onDispatch () 方法，该方法的动作有：

- \1. 找出在本服务类中与请求数据相对应的函数；
- \2. 解码请求数据中的函数参数；
- \3. 执行 XXXServantImp 类中用户定义的相应 RPC 方法；
- \4. 编码函数执行后的结果；
- 5.return tars::TARSSERVERSUCCESS。

上述步骤是按照默认的服务端自动回复的思路去阐述，在实际中，用户可以关闭自动回复功能（如：current->setResponse (false)），并自行发送回复（如：servant->async_response_XXXAsync (current, ret, rStr)）。到此，服务端已经执行了 RPC 方法，下面稍微总结一下本小节的内容：

![图（2-27）处理TARS协议的请求](.\picture\up-af0d30ce6dc0214da976851b105632c0520.png)

#### 3. 将响应数据包 push 到线程安全队列中并通知网络线程

处理完 RPC 请求，执行完 RPC 方法之后，需要将结果（下面代码中的 buffer）回送给客户端：

```
void ServantHandle::handleTarsProtocol(const TarsCurrentPtr &current)
{
    // 1-对请求上下文current进行预处理
    // 2-寻找合适的服务servant
    //3-业务逻辑处理
//回送响应，本节分析
    if (current->isResponse())
    {
        current->sendResponse(ret, buffer, TarsCurrent::TARS_STATUS(), sResultDesc);
    }
}
```

由于业务与网络是独立开来的，网络线程收到请求包之后利用条件变量来通知业务线程，而业务线程才有什么方式来通知网络线程呢？由前面可知，网络线程是阻塞在 epoll 中的，因此需要利用 epoll 来通知网络线程。这次先看图解总结，再分析代码：

![图（2-28）数据push到队列中并通知网络线程](.\picture\up-421233cb43561cd5b2d250d5b94945b26c7.png)

在 ServantHandle::handleTarsProtocol () 中，最后的一步就是回送响应包。数据包的回送经历的步骤是：编码响应信息 —— 找出与接收请求信息的网络线程，因为我们需要通知他来干活 —— 将响应包放进该网络线程的发送队列 —— 利用 epoll 的特性唤醒网络线程，我们重点看看 NetThread::send ():

```
void TC_EpollServer::NetThread::send(uint32_t uid, const string &s, const string &ip, uint16_t port)
{
    if(_bTerminate)
    {
        return;
    }
 
    tagSendData* send = new tagSendData();
 
    send->uid = uid;
    send->cmd = 's';
    send->buffer = s;
    send->ip = ip;
send->port = port;
 
    _sbuffer.push_back(send);
 
    //通知epoll响应, 有数据要发送
    _epoller.mod(_notify.getfd(), H64(ET_NOTIFY), EPOLLOUT);
}
```

到此，服务器中的业务模块已经完成他的使命，后续将响应数据发给客户端是网络模块的工作了。

### 发送 RPC 响应

获取了请求，当然需要回复响应，从上面知道业务模块通过_epoller.mod (_notify.getfd (), H64 (ET_NOTIFY), EPOLLOUT) 通知网络线程的，再加上之前分析 “2.3.1 接受客户端请连接” 以及 “2.3.2 接收 RPC 请求” 的经验，我们知道，这里必须从 NetThread::run () 开始讲起，而且是进入 case ET_NOTIFY 分支：

```
try
{
    const epoll_event &ev = _epoller.get(i);
    uint32_t h = ev.data.u64 >> 32;
 
    switch(h)
    {
    case ET_LISTEN:
        ……
        break;
    case ET_CLOSE:
        //关闭请求
        break;
    case ET_NOTIFY:
        //发送通知
        processPipe();
        break;
     case ET_NET:
        //网络请求
        ……
        break;
      default:
         assert(true);
      }
}
```

在 NetThread::processPipe () 中，先从线程安全队列中取响应信息包：_sBufQueue.dequeue (sendp, false)，这里与 “2.3.3 处理 RPC 请求” 的第 3 小点 “将响应数据包 push 到线程安全队列中并通知网络线程” 遥相呼应。然后从响应信息中取得与请求信息相对应的那个 Connection 的 uid，利用 uid 获取 Connection：Connection *cPtr = getConnectionPtr (sendp->uid)。由于 Connection 是聚合了 TC_Socket 的，后续通过 Connection 将响应数据回送给客户端，具体流程如下图所示：

![图（2-29）服务端向客户端返回响应数据](.\picture\up-1778400472484197fc382d51fc431b7513b.png)

### 服务端工作总结

这里用图解总结一下服务端的工作过程：

![图（2-30）服务端工作图](.\picture\up-3e365ca21c58e76602e6b5915e5f648c234.png)

### 总结

TARS 可以在考虑到易用性和高性能的同时快速构建系统并自动生成代码，帮助开发人员和企业以微服务的方式快速构建自己稳定可靠的分布式应用，从而令开发人员只关注业务逻辑，提高运营效率。多语言、敏捷研发、高可用和高效运营的特性使 TARS 成为企业级产品。

《微服务开源框架 TARS 的 RPC 源码解析》系列文章分上下两篇，对 RPC 调用部分进行源码解析。本文是下篇，我们带大家了解了一下 TARS 的服务端。欢迎阅读上篇《初识 TARS C++ 客户端》