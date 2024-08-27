# taf/tars框架学习

官方文档：https://doc.tarsyun.com/ 镜像



## 概念

**应用-服务-接口**

使用Tars框架的服务，其的服务名称有三个部分：

APP： 应用名，标识一组服务的一个小集合，在Tars系统中，应用名必须唯一。例如：TestApp；

Server： 服务名，提供服务的进程名称，Server名字根据业务服务功能命名，一般命名为：XXServer，例如HelloServer；

Servant：服务者，提供具体服务的接口或实例。例如:HelloImp；

说明：

一个Server可以包含多个Servant，系统会使用服务的App + Server + Servant，进行组合，来定义服务在系统中的路由名称，称为路由Obj，其名称在整个系统中必须是唯一的，以便在对外服务时，能唯一标识自身。

因此在定义APP时，需要注意APP的唯一性。

例如：TestApp.HelloServer.HelloObj。

一个应用包含多个服务，一个服务包含多个接口





![image-20240809163334199](D:\UserData\18122252978\AppData\Roaming\Typora\typora-user-images\image-20240809163334199.png)

- tarsRegistryServer -- 类似于DNS，管理名字
- NotifyServer  -- 展示在web，也可以发短信
- StatServer  -- 所有服务的流量等信息，写入数据库
- PatchServer -- 发布服务，必须和web部署在一起
- ConfigServer -- 存在DB，addconfig可以将db里的配置拉到本地
- propertyServer -- 脱离于框架，服务独有的信息



![image-20240809165401537](D:\UserData\18122252978\AppData\Roaming\Typora\typora-user-images\image-20240809165401537.png)





![image-20240809170123627](D:\UserData\18122252978\AppData\Roaming\Typora\typora-user-images\image-20240809170123627.png)





## 离线安装Web

#### 依赖安装

其实就是模拟`linux-install.sh`文件

frame框架依赖node、nvm、pm2

1. 安装nvm (可以不安装)

   官网下载地址： https://github.com/nvm-sh/nvm/releases

   

2. 安装node

   官网下载地址： https://nodejs.org/en/download/prebuilt-binaries

   直接下载编译过的，

   

3. 安装pm2

   只能从另一台服务器安装完成后，复制过来
   默认安装/usr/local/node/lib/node_modules/，复制到本地node路径

   

4. 修改环境变量

​	vim /ect/procfile 修改全全局的变量

​	最后一行添加	

```bash
export NVM_DIR="$HOME/.nvm/nvm-0.38.0"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

​	

#### 一键部署

https://doc.tarsyun.com/#/installation/source.md

根据官网内容使用**linux-install.sh**脚本。



**在其他机器里运行脚本后，复制node_modules目录（node下载的依赖）到 web**目录下，再使用脚本



:warning:**注意在部署web的时候，需要进入到/usr/local/tars/cpp/deploy/标准目录，不要在自己的目录里用脚本部署，会导致模块部署错误**

 https://doc.tarsyun.com/#/dev/tarscpp/tarscpp.md





## 创建服务

### 创建Tars服务

https://doc.tarsyun.com/#/dev/tarscpp/tarscpp.md

在**管理系统上的部署**暂时先到这里，到此为止，只是使你的服务在管理系统上占了个位置，**真实程序尚未发布**。

#### 生成服务

使用cmake脚本cmake_tars_server.sh的生成服务：

<Usage:  ./cmake_tars_server.sh  App  Server  Servant>

```shell
$ ./cmake_tars_server.sh  App  Server  Servant
```

对应web端的[运维管理-部署申请]中的

- App--应用栏,填写前可以在[运维管理-应用管理]中增加
- Server--服务名称栏
- Servant--会自动增加Obj尾缀，对应OBJ栏（web填写时需要加上Obj）



#### **发布操作**

把服务发布出去，可以供其他app使用



运维方式：**服务管理-发布管理-选中-上传发布包-上传**



开发方式：

- make release 发布所有 并生成h、tars、mk文件
- make upload 上传所有
- make tar 压缩
- **make <Server Name>-<Command> 指定一个服务**，例make HelloServer-upload 上传HelloServer
- make <Server Name>-tars-merge 可以将许多服务merge成一个



注意：

1. 上传地址，可以通过cmake修改**TARS_WEB_HOST**变量来控制
2. {"ret_code":500,"err_msg":"您还没有登录","data":{}} 可以在web端增加一个token用作上传，[用户中心-Token管理-新增Token] 同样cmake -D**TARS_TOKEN** 来修改值
3. 两种方式任选一种即可
4. release发布的实质就是把.h复制到共享目录，供其他服务等使用

   ```shell
   [root@centos72 build]# make HelloServer-release
   [ 20%] Built target tars-HelloServer
   [ 80%] Built target HelloServer
   Scanning dependencies of target HelloServer-release
   [100%] call /usr/local/tars/cpp/script/HelloServer/build/src/run-release-HelloServer.cmake
   cp -rf /usr/local/tars/cpp/script/HelloServer/src/Hello.h /home/tarsproto/TestApp/HelloServer
   cp -rf /usr/local/tars/cpp/script/HelloServer/src/Hello.tars /home/tarsproto/TestApp/HelloServer
   ```




#### 配置方式

1. Web端：【服务管理-服务配置】 添加配置

2. 服务源代码：在初始化函数内增加读配置函数

   ```cmake
   Add_Config
   ```

   

#### 日志方式

```cmake
LOG->debug() <<"xxx"<endl;
TLOG_DEBUG("xxx");
TLOGDEBUG("xxx")
```

日志执行效率从上往下递增，但是`TLOGDEBUG`方式不会打印函数名



采用 tars2cpp 工具自动生成 c++文件：/usr/local/tars/cpp/tools/tars2cpp hello.tars 会生成 hello.h 文件



:warning:

upload时会将.tgz的内容拷贝到/usr/local/app/tars/tarsnode/data/TestApp.HelloServer/bin/HelloServer

/usr/local/app/tars/tarsnode/data/tmp目录下会记录每一次upload,所以每次upload之前需要**make tar**一下！



### tars 接口文件

xx.tars可以展示模块-接口 及其关系



### 创建http服务

非tars协议，

函数继承关系如下：

|              | tars协议服务                                                 | 非tars协议服务                                               |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| obj          | 会生成 <objName>.h(协议file)                                 | 不会生成.h文件                                               |
| xxImp.h/.cpp | 实现类继承<objName>                                          | 实现类继承Servant                                            |
| xxServer.cpp | 继承框架Application                                          | `addServantProtocol(ServerConfig::Application + "." + ServerConfig::ServerName + ".httpObj", &TC_NetWorkBuffer::parseHttp);` |
| <objName>.h  | 用<objName>Proxy继承实现类，把代理类的名字装入auto指针def为<objName>Prx，都在<AppName>的命名空间里 | 无                                                           |

实际<objName>也是继承的Servant，实现doRequest而已

对于非tars协议，我们也可以通过`addServantProtocol`添加**parse<Proxy>**协议解析器，对协议进行解析。协议解析器吐出来的包交给doRequest(ptr,buffer)进行响应



Servants管理里可以更改tars配置，上文所说的配置是服务配置



| servant AdminObj这个是供web端监控的ip：port | endpoint tcp -h 127.0.0.1 -p 8445 -t 10000 |
| ------------------------------------------- | ------------------------------------------ |
|                                             |                                            |



#### 服务之间互相调用

调用步骤

1. include头文件
2. 对象定义
3. 对象初始化！ 使用通讯器Application::getCommunicator()->stringToProxy<TestApp::HelloPrx>("TestApp.HelloServer.HelloObj");产生两个服务的链接
4. 使用对象调用函数



注：

- 通讯器Communicator管理客户端所需资源的组件
- 独立客户端 需要new Communicator
- 服务间调用，无需new 直接用单例Communicator即可





//TODO:看httpdemo，写一个通过callback回调方式实现异步响应的服务



//问题：std::list<std::shared_ptr<Buffer>> _bufferList;为什么TC_NetWorkBuffer类的 buffer要用list包裹起来

//TODO：尝试！tars obj拥有容灾能力，无需分组，只需要使用冒号在连接服务的参数那增加上备份服务的ip：port即可.

例：`TestApp.TestServer.TestObj@tcp -h 127.0.0.1 -t 60000 -p 12345 ` **`:tcp -h 188.190.12.92 -t 60000 -p 12345`**

//TODO  查看框架代码异步调用async、回调是如何实现的

A调用B，B调用C  current参数set_response =false 来实现B的全异步。b不回包给a，等待c回包后再回包a



##### 主要目录框架服务目录: 

/usr/local/app/tars/xxx业务服务:  /usr/local/app/tars/tarsnode/data/服务日志:  /usr/local /app/tars/app_log/xxx/xxxserver/*.logWeb日志: /usr/local/app/web/log

---

### 请求服务

##### 直连方式

```c++
static string helloObj = "TestApp.HelloServer.HelloObj@tcp -172.16.0.2 -t 60000 -p 25280";
_comm = new Communicator();
HelloPrx pPrx = _comm->stringToProxy<HelloPrx>(helloObj);
```



##### ⭐请求主控服务，查询所需服务

优势：这种请求方式，在服务器迁移时，无需大量修改直连地址，只需要将主控地址进行控制即可，其他交给tarsregistry完成查询

```c++
static string helloObj = "TestApp.HelloServer.HelloObj";
_comm = new Communicator();
#locator 表示设置主控信息，后面填查询接口名@地址
_comm->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h 172.16.0.2 -t 60000 -p 17890");
#请求流控控制服务，填写主控地址后无需再写stat地址，统一交给stat管理
_comm->setProperty("stat","tars.tarsstat.StatObj");
HelloPrx pPrx = _comm->stringToProxy<HelloPrx>(helloObj);
```



注意：

- `new Communicator();`这种方式才需要去请求框架的服务，且这种方式仅能主控该接口的服务，其他框架服务（如stat）都需要额外配置，而直接使用内置Communicator是无需设置的。
- 主控Query服务默认端口17890



## JMem 学习





## 相关文章

微服务开源框架TARS的RPC源码解析 之 初识TARS C++服务端：

https://cloud.tencent.com/developer/article/1663146





---

### tars接口协议

即xx.tars文件的协议

查看**tars-tup.md**

