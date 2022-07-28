# git 环境
**密钥生成**
```sh
 ssh-keygen -t rsa -b 4096 -C "demo@test.com"

#  查看公钥
cat $HOME/.ssh/id_rsa.pub

# 查看是否成功添加到github
ssh -T git@github.com
```

**git仓库在Fork后同步源仓库**

```sh
# 1. 列出当前为fork 配置的远程仓库

git remote -v

# 如果只有origin的两行,比如只有origin说明只有一个fork后的Repo.  
# 说明你未设置 upstream （中文叫：上游代码库）一般情况下，设置好一次 upstream 后就无需重复设置。
# origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
# origin  https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)

# 指定新的远程上游仓库
git remote add upstream https://github.com/xxx/xxx.git

# 检查一下
git remote -v

# origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (fetch)
# origin    https://github.com/YOUR_USERNAME/YOUR_FORK.git (push)
# upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (fetch)
# upstream  https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY.git (push)


# 1.从上游仓库获取分支及其各自的提交，传送到本地，对 master 分支的提交将存储在本地分支 upstream/master 中。

git fetch upstream
# remote: Counting objects: 75, done.
# remote: Compressing objects: 100% (53/53), done.
# remote: Total 62 (delta 27), reused 44 (delta 9)
# Unpacking objects: 100% (62/62), done.
# From https://github.com/ORIGINAL_OWNER/ORIGINAL_REPOSITORY
#  * [new branch]      master     -> upstream/master

# 2.切换到本地的主分支。

git checkout master
# Switched to branch 'master'
# 将来自 upstream/master 的更改合并到本地 master 分支中。这就实现了与上游仓库的同步，而不会丢失本地的更改。


git merge upstream/master
# Updating a422352..5fdff0f
# Fast-forward
#  README                    |    9 -------
#  README.md                 |    7 ++++++
#  2 files changed, 7 insertions(+), 9 deletions(-)
#  delete mode 100644 README
#  create mode 100644 README.md
# 最后推送到远程仓库就完成了。

git push origin master

# gitlab 界面fork 仓库的界面会出现 Create merge request 的请求按钮，在本地的仓库提MR 到远程仓库和分支

# 点击之后，Change breanchs , 选择当前的仓库分支 以及选择要合并的对应的上游仓库的分支，进行合并即可

在origin 仓库提MR 到 上游仓库
```

### git 回滚操作
```sh
# 回滚到上一个版本
git reset --hard HEAD^

# 强制推送到远程分支 更新仓库
git push -f origin v2.25.1.0
```