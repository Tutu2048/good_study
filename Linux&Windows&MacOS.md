# Linux&Windows&MacOS

## Linux

#### Shell命令

##### objdump	 

Display information from object <file(s)>.

Usage: objdump <option(s)> <file(s)>

```shell
$objdump -T xx.a	#可以查看静态库是以什么架构编译的 
```

对于查看目标文件和可执行文件的**符号**而言，objdump -T <xx> 和nm <xx> 是同样的效果,可以查看是否编译并**加载了一些函数**供他人使用

---

##### touch

Usage: touch [OPTION]... FILE...

不光可以创建空的文件，也可以对于已存在的数据，**更新其修改时间**为现在时间

---

##### pkgconfig

帮助管理系统编译和链接期间库依赖的工具

```shell
# 查询库的编译标志
$pkg-config --cflags glib-2.0

# 查询库的链接标志
$pkg-config --libs glib-2.0

# 查询库的编译和链接标志，以及版本要求
$pkg-config --cflags --libs glib-2.0 >= 2.40
```

怎么让源码安装的库被pkg-config找到？

**.pc文件**

在源码安装库时，make install时会**将生成的xx.pc文件复制到库文件认为的目录下**，比如`/usr/local/lib/pkgconfig`，有时是正确的，那就会被pkgconfig所找到。若没有可以设置环境变量

`export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH`

或者在安装前，查看其他.pc文件在哪，指定安装路径

`./configure --with-pkgconfigdir=/path/to/extra/lib/pkgconfig`（未成功）

---

##### ldd

`ldd` 是一个在 Unix 和类 Unix 系统（包括 Linux 和 macOS）中用于打印出动态链接的共享库的列表的命令行工具。使用 `ldd` 可以查看一个可执行文件或库在运行时所依赖的所有共享库，以及这些库的查找路径和状态。

以下是一些使用 `ldd` 的基本命令和选项：

1. **查看可执行文件的依赖**：

   ```
   ldd /path/to/executable
   ```

2. **查看库文件的依赖**： 也可以对库文件使用 `ldd` 来查看它们所依赖的其他库。

   ```
   ldd /path/to/library.so
   ```

3. **输出所有信息**： 默认情况下，`ldd` 只显示需要加载的库。使用 `-v`（或 `--verbose`）选项可以显示所有信息，包括已加载和未加载的库。

   ```
   ldd -v /path/to/executable
   ```

4. **不打印路径**： 使用 `-x` 选项可以告诉 `ldd` 不打印库的完整路径，只显示库的名称。

   ```
   ldd -x /path/to/executable
   ```

5. **打印依赖树**： 使用 `-t` 选项可以打印出库的依赖树，显示它们之间的依赖关系。

   ```
   ldd -t /path/to/executable
   ```

6. **检查环境变量**： `ldd` 会考虑 `LD_LIBRARY_PATH` 环境变量来查找库。如果需要忽略这个环境变量，可以使用 `-D` 选项。

   ```
   ldd -D /path/to/executable
   ```

`ldd` 的输出通常包括以下几列：

- **库名称**：依赖库的名称或路径。
- **库类型**：例如，`lib` 表示是一个库，`not found` 表示库没有被找到。
- **库状态**

---

##### pgrep

`pgrep <program name>`根据程序名查找进程id

---

##### **sed** 

```shell
#Usage: sed [OPTION]... {script-only-if-no-other-script} [input-file]...
#option -i 直接编辑文件本身，不加-i编辑输出文本
$sed [-i] 's/old/new/' file.txt#substitute
$sed [-i] '3s/old/new/' file.txt
$sed [-i] 'i\new line' file.txt#insert
$sed [-i] '3a\new line' file.txt#append
$sed [-i] '$a\new line' file.txt
$sed  -n '3p' file.txt
$sed -n '1,5p' file.txt
```

##### 管道｜

- **xargs**

  cmd1 | xargs cmd2    xargs(extended arguments) 可以将标准输入里面的每一行转换成命令的参数     xargs往往和管道一起使用，

  ```shell
  find . -name "*.c" | xargs grep -nE "\<main\("
  ```

---

##### grep

- `-n`：在输出中显示匹配行的行号。
- `-E`：将模式解释为扩展正则表达式（这允许使用更复杂的正则表达式语法）。

---

##### 重定向>

- `> file` ：清空file 重写内容

- `>> file`：添加内容在file末尾



---

#### 挂载的一系列操作

当你开开心心买了一块硬盘，想给自己的linux服务器扩容一下，却发现不会，那不是很尴尬。解决这个麻烦只需要：

硬盘、磁盘是存储01数据的硬件，直接将硬盘拿给程序员，它相当于一块不太结实的板砖。二进制的数据并不适合阅读，此时需要一个“翻译官”---操作系统的驱动程序，让我们重新认识一下这块硬盘。

##### fdisk 

​	（find disk）通过`$fdisk -l`该命令，找到这块新买的硬盘。对于linux服务器来说，一切皆文件，当硬盘通过驱动程序展现在操作系统上就是一个在/dev目录下的一个文件。

当你的硬盘足够大，且你想存放各样的数据时，你可以通过分区的方式，对不同的数据进行分类管理。

##### df 

​	`df -l`查看分区。新硬盘未分区，可以通过`fdisk /dev/vdb` ，根据提示进行分区。

而当你开开心心想要将存放一张照片的时候，突然想起：

##### blkid、mkfs

​	输入了一下`$blkid`，发现分区都是TYPE=“ext3”，可你不想用这个系统类型，这时候就需要用`$mkfs.xfs /dev/vdb1`命令把你需要分区修改成xfs系统类型，`mkfs.[typename]`

ok，接下来就可以用了吗？不是的，上述一切的操作，就相当于网购下单前，一系列对比、挑选

##### mount

​	而mount命令，就相当于下单了。`$mkdir /mnt/storage`创建一个目录，作为你快递的地址，mount可不会给你创造一个家，家是由你自己创造的。`$mount /dev/vdb1 /mnt/storage/`将分区挂载到目录下。这时候就可以正常使用硬盘的存储功能了。

​	但是，还有一个问题当你使用完之后，第二天发现，自己的磁盘又不在/mnt/storage下了，这是因为少了一步--自动挂载。

`vim /etc/fstab`打开挂载的配置文件，添加上下面那一句

`UUID=3dc34765-09e2-43f5-aa90-64b7949edc68 /mnt/storage xfs defaults 0 0`

uuid可以通过blkid获得，这样就可以正常使用了。

当然，mount对于一切皆文件的linux来说，并不是只能挂载硬盘，还可以通过samba，获取小伙伴的共享文件夹。

`$mount -t cifs -o username=root,password=Pass //172.192.22.1/shared /home/xxShared`

- `-t cifs`: 指定文件系统类型为 CIFS。
- `-o`: 后面跟的是挂载选项，可以有多个，用逗号分隔。
- `//172.192.22.1/shared `: CIFS 服务器的网络路径，`172.192.22.1` 是服务器的 IP 地址，`shared ` 是共享的名称。
- `/home/xxShared`: 本地系统上作为挂载点的目录。

注：CIFS（Common Internet File System，即 SMB/Microsoft 网络文件共享

---

**find  . -atime [n]**

+n     for greater than n,

-n     for less than n,

n      for exactly n.

:warning:这边的大于，单位是天。也就是说 **+1 至少是2天，而不是大于1天**

```shell
   -atime n
          File  was  last  accessed n*24 hours ago.  When find figures out how many 24-hour periods ago
          the file was last accessed, any fractional part is ignored, so to match -atime +1, a file has
          to have been accessed at least two days ago.
```

---

##### **Bash shell 的配置文件**

`/etc/profile` 和 `~/.bashrc` 都是 Bash shell 的配置文件，但它们的作用和使用场景有所不同：

1. **`/etc/profile`**：
   - 这是一个全局的配置文件，对系统中所有的用户都生效。
   - 当任何用户登录系统时（包括通过终端或者图形界面），这个文件都会被执行。
   - 它通常用于设置环境变量、定义系统级的功能和别名等。
   - `/etc/profile` 也可以调用其他配置文件，如 `/etc/profile.d/` 目录下的脚本。
2. **`~/.bashrc`**：
   - 这是一个用户的配置文件，只对当前用户生效。
   - 当用户启动一个新的交互式 shell 时，比如通过打开一个新的终端窗口，`.bashrc` 文件会被执行。
   - 它通常用于设置用户级别的环境变量、定义别名、函数、bash选项等。
   - `~/.bashrc` 可以包含针对单个用户的个性化配置。
3. **关系**：
   - `~/.bashrc` 通常会调用 `/etc/bash.bashrc`，后者可以包含一些全局的配置，但不会覆盖用户在 `~/.bashrc` 中的设置。
   - 系统管理员可以通过修改 `/etc/profile` 和 `/etc/bash.bashrc` 来影响所有用户的 shell 环境。
   - 用户可以通过修改自己的 `~/.bashrc` 来定制自己的 shell 环境。
4. **使用场景**：
   - 如果你需要设置一些对所有用户都生效的环境变量或别名，你应该在 `/etc/profile` 中设置。
   - 如果你需要为单个用户设置环境变量或别名，或者添加一些仅对单个用户生效的配置，你应该在 `~/.bashrc` 中设置。
5. **注意**：
   - 在 `~/.bashrc` 中设置的环境变量可能不会在非交互式 shell 中生效，比如在脚本中。如果需要在脚本中使用这些环境变量，你可能需要在脚本中显式地设置它们，或者使用 `source` 命令来加载 `~/.bashrc`。
6. **继承关系**：
   - 当你登录系统时，首先执行 `/etc/profile`，然后执行 `~/.bashrc`。但是，如果你在某个脚本中显式地调用了 `source ~/.bashrc`，则会按照调用的顺序执行。



---

##### dos2unix

linux文件出现^M Windows的回车，可以通过dos2unix filename 转化为linux的回车

---

##### **chown**

Usage: chown [OPTION]... [ OWNER] [:[GROUP]] FILE...

仅root用户有权限进行授权，可以将文件授权给其他用户所有，其他用户可以拥有所有权

---

##### 重定向

重定向输出到文件时，通常使用`>`和`>>`这两个操作符。`>`用于将输出重定向到文件中，如果文件已存在，则会覆盖原有内容；**`>>`用于将输出追加到文件末尾**，如果文件已存在，不会覆盖原有内容，而是添加到文件内容的后面。

---

#### Linux app管理

##### 把程序交由到**systemd**管理

```shell
$ cd /etc/systemd/system/xx.service	#link to /usr/lib/systemd/system/xx.service
$ cd /usr/lib/systemd/system/xx.service #real service
#添加、修改该两个文件即可被systemd找到
$systemctl daemon-reload	#修改完重新加载一下文件
$systemctl status/start/enable/stop/restart xx  #默认省略service；enable开启自启动
```

---

​										<!--Chapters End-->

---

## Windows

##### Q：为什么编译DLL时总会同时生成一个LIB？

**A：**在Windows平台上，编译动态链接库（DLL）时生成的 `.lib` 文件被称为 **导入库**（import library）。它的作用是帮助链接器在编译和链接其他使用该DLL的程序时，找到DLL中导出的函数和变量。

- **导入库（`.lib` 文件）** 是在**编译和链接时**使用的，它包含DLL中导出函数和变量的符号信息和占位符。
- 是特殊的静态文件，仅用于程序员编译、调试时使用

---

​										<!--Chapters End-->

---

## MacOS

> 键盘设置 Command == Alt ; Win ==Option ;Ctrl ==Control

##### 快速查词

##### **方法一：**

1. command + space 打开聚焦  
2. 输入需要查的词 
3. command +D

 即可打开词典并搜索该词

**方法二：**

利用触控板 三指查词

**方法三：**

选中词 command + Option + D  ，在大部分App内有用

