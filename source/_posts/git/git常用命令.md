---
title: git常用命令
categories:
  - GIT
tags:
  - 规范
  - GIT
  - 常用命令
abbrlink: 423abe9e
date: 2025-02-06 14:54:00
---

> 在Windows平台需要在`git bash`中进行操作，mac直接在终端操作

git文件的四种状态：
- 未跟踪
- 未修改
- 已修改
- 暂存

## 基础配置

``` powershell
$ git config --global user.name "nick.zhu"
$ git config --global user.email nick.zhu@shaiyang.cn
# 查看配置
$ git config --list
```

## 创建仓库

``` powershell
$ git init
```

## 克隆仓库/clone

``` powershell
$ git clone xxx.git
```

## 文件跟踪

``` powershell
$ git add <name>
# 跟踪当前目录所有文件
$ git add .
# 删除跟踪
$ git rm <name>
# 删除跟踪并保留在目录
$ git rm --cache <name>
```

## 缓存

``` powershell
# 设置修改过的文件为缓存状态
$ git add <file-name>
# 取消文件缓存状态
$ git reset HEAD <name>
```

## 提交

提交遵守[约定式提交](https://www.conventionalcommits.org/en/v1.0.0/)，使用Angular规范

``` powershell
<type>[optional scope]: <description>
// 空行
[optional body]
// 空行
[optional footer(s)]
```

``` powershell
$ git commit
$ git commit -m <message>
# 取消提交（无法撤销第一次，从第二次开始可以撤销）,--soft选项只取消提交，缓存还在。--hard取消提交和缓存
# 取消当前提交
$ git reset head --soft
# 取消上一次提交
$ git reset head~ --soft
# 取消倒数第二次提交
$ git reset head~2 --soft
```

### 提交内容前缀/type

``` json
{ "Features","新增"},
{ "feat","新增"},
{ "fix","修复 "},
{ "Fixed","修复 "},
{ "Changed","变更 "},
{ "changed","变更 "},
{ "docs","文档 "},
{ "Refactored","优化"},
{ "refactor","优化"},
{ "Deprecated","即将删除"},
{ "deprecated","即将删除"},
{ "Removed","删除"},
{ "removed","删除"},
{ "BREAKING CHANGE","破坏性变更"},
```

## 标签/tag

使用[semver](https://semver.org/lang/zh-CN/)版本定义

``` powershell
# 列出所有标签
$ git tag
# 新建tag
$ git tag v0.3.2
# 使用-a参数新建一个带备注的tag，备注信息使用-m指定
$ git tag -a v0.3.2 -m "version 0.3.2"
# 显示tag详细信息
$ git show v0.3.2
# 将tag同步到远程服务器
$ git push origin v0.3.2
# 本地删除
$ git tag -d v0.3.2
# 远程删除
$ git push origin :refs/tags/v0.3.2
```

## 状态

``` powershell
# 查看当前仓库状态，也可以查看冲突文件
$ git status
# 查看变更明细
$ git diff
# 查看历史提交
$ git log
# 美化输出
$ git log --pretty=oneline
# 自定义输出
# %h	简化哈希
# %an	作者名字
# %ar	修订日期（距今）
# %ad	修订日期
# %s	提交说明
$ git log --pretty=format:"%h - %an,%ar:%s"
# 图形化查看提交日志
$ git log --graph
# 查看所有分支提交日志
$ git log --all
```

## 远程仓库/remote

``` powershell
# 链接一个叫 orgin 的远程仓库
$ git remote add orgin https://xxx.git
# 对仓库名称重命名
$ git remote rename orgin origin
# 将master分支推送到 远程仓库
$ git push origin master
# 可以通过 git push -u origin master提交的方式，提交过一次之后，以后提交可以简写git push
$ git push -u origin master
$ git push
```

## SSH鉴权（替代token鉴权）

``` powershell
$ cd ~/.ssh
# 生成一个密钥 -t 加密方式 -b 大小 -C 评论(github推荐填写邮箱)
$ ssh-keygen -t rsa -b 4096 -C "nick.zhu@shaiyang.cn"
# 输入证书名称和密码生成相应证书，将公钥配置到git仓库（github:settings/ssh/new）
$ cat xxx.pub
```

## 分支/branch

``` powershell
# 可以通过git log查看当前所在分支
# 也可以通过git branch --list 查看
$ git branch --list
# 列出所有远程分支
$ git branch -r
# 新建一个分支，但依然停留在当前分支
$ git branch [branch-name]
# 新建一个分支，并切换到该分支
$ git checkout -b [branch]
# 切换分支
$ git checkout feature1
# 修改后提交 -a 提交所有暂存文件 -m 提交说明，可以简写 -am
$ git commit -am 'featrue1'
# 合并指定分支到当前分支
$ git merge [branch]
# 删除分支
$ git branch -d [branch-name]
# 删除远程分支
$ git push origin --delete [branch-name]
$ git branch -dr [remote/branch]
# 拉取远程分支
$ git fetch
```

## 储藏/stash

``` powershell
# 比如代码写到一般需要切换分支，可以通过git stash进行代码储藏
$ git stash
# 恢复之前存储的内容
$ git stash apply
# 查看存储记录
$ git stash list
# 按照序号恢复存储，0是最后一次存储
$ git stash apply stash@{2}
```
