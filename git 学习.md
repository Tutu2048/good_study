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
