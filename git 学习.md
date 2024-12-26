删除本地、远程分支

```bash
git branch -d xxx
git push origin -d xxx
```



更新远程引用的信息，但不合并或拉取任何代码。

> 用于查看远端变化

```bash
git remote update
```



通过token的方式登录，是账户级别的，类似于tmp user 账号 setting -Developer Settings -Personal access tokens (classic)

通过ssh key public、private key 的方式是针对仓库的，仓库 settings Deploy keys



这是最安全的方法，因为它不会改变项目的历史。它会创建**一个新的提交**，这个提交是被撤销提交的相反操作。

```bash
git revert <commit-hash>
```



```bash
git reset --soft <commit-hash>^
git reset --mixed <commit-hash>^
git reset --hard <commit-hash>^
```

**软重置（Soft Reset）**：参数不会影响索引文件或工作树。它只是将 HEAD 指向你指定的提交。如果你想保留更改并重新提交，这是一个有用的选项。撤销的commit的内容在**暂存区**

**混合重置（Mixed Reset）**：是默认模式，它将重置索引以匹配指定提交，但不会更改工作树。如果你想**撤销 *git add* 的操作**，把内容放回到工作区了。

**硬重置（Hard Reset）**：撤销提交，并且丢弃所有改动。



**解决冲突**

打开冲突文件，完成后

```bash
git add conflictFile 
git commit -m "Resolve conflict"
```

会出现在`merge`和`rebase`中



**rebase**

1、以某次commit 或者某个分支为基础，开始整理分支线

```bash
git rebase [基hash、基分支]
```

2、出现冲突、解决冲突

3、结束后，继续变基

```bash
git rebase --continue
```

4、由于变基会改变提交历史，所以push时需要--force

```bash
git push --force-with-lease origin <branch-name>
git push --force origin <branch-name>
```

更推荐--force-with-lease
