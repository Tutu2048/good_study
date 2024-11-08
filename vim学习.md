# vim

vim `：command`

- `:e filename` ，打开一个文件，作为缓冲区
- `:bd`，关闭当前缓冲区
- `tab`，切换缓冲区

---对比---

- `:vsp filename` ,（vertical split）打开一个文件，作为一个窗口
- `:q` ,退出该窗口
- `<crl> +w+w`,切换窗口

**tips**：vim中， `：command`除了可以使用vim内置的命令，还可以**使用`:cd ..`等shell命令**，当然很多展示的shell命令是不兼容的，例如`:cat`



##### 常用编辑

- `guu`整行小写,

  可视模式（v）下，u选中文本小写

- `gUU`整行大写，

  可视模式（v）下，U选中文本大写

  



##### 常用删除

注：vim常用命令模式，commmad+target，`di +括号`，对于这些被包括起来的内容的删除，会比`df+target`好用

`di}`;`di]`;`di)`; `di"`; `di'`  ;`diw`

- `d`：删除命令
- `i`：进入插入模式。
- `w`：移动到下一个单词的开头。

注：把d换成c (change)命令，可以直接进入编辑模式



##### 执行外部命令行操作

- `:!+shell command`



#### vimplus

默认以**,逗号**作为<leader>键

`,h`可以查看快捷键帮助

`,n`开关任务管理器



- `zo`：打开被折叠的代码。“z”象形or zone open
- `zc`：关闭被折叠的代码。close
