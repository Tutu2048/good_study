## Clion初级学习

https://zhuanlan.zhihu.com/p/97175720

##### 一、Clion安装

...

##### 二、Cygwin安装

为了在window下的cmd中用到linux上常用的grep，find等命令。

```bash
//映射关系
'./.bashrc' -> '/home/18122252978//.bashrc'
'./.bash_profile' -> '/home/18122252978//.bash_profile'
'./.inputrc' -> '/home/18122252978//.inputrc'
'./.profile' -> '/home/18122252978//.profile'
```



添加环境

```shell
path C:\cygwin64\bin
```

##### 三、tafjce同步





#### 四、快捷键

双击shift ---搜索



##### 五、本地开发流程

1、工具链Toolchains

​	cmake-编译器-调试器

2、环境配置 cmake里

3、auto代码

i)Editor-File and Code Templates里面添加，author time ...
	Scheme:  1.default 2.project仅当前项目生效

ii)Editor-Live Templates
	关键字+tab->拓展成设置

iii)类方法生成generate
	ctrl+insert

tips：ctrl+f6、shift+f6 批量修改函数

4、代码规范-代码风格
	ctrl+alt+L ->Reformat Code	大驼峰小驼峰，变量命名规范，缩进...

5、编码完成运行
	Edit Configurations

6、debug
	evaluate 变量、表达式计算器

7、性能分析--Flame Graph火焰图
https://www.jetbrains.com/help/clion/cpu-profiler.html `profiler分析器安装`



##### 六、远程开发

1. 可以配置启动文件Edit Configurations -> **docker** 配置远程、启动远程服务

2. 远程编译链配置

   版本不对时：remote cmake gdb 可以去/usr/local/bin里面找确定的（可能有多个版本）

3. 远程CMake(一个cmake对应一个中间文件夹)

   记得需要将远程服务器的环境变量进行添加`$printenv`

4. 远程代码部署-同步代码
   Deployment-Mappings  
   Local path->本地需要同步到远程的文件夹（一般为src文件夹）
   Deployment path -> 同步到远程的哪个文件夹中

5. 远程编译+运行

6. 远程调试！
   Edit Configurations      gdbsever：[port]  

七、git版本控制

八、插件推荐
	主题类：theme自己在Marketplace搜
	.ignore : ♥git忽略模板
	Docker
	CMakePlus ：付费
	

​	