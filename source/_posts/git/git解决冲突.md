---
title: git解决冲突
categories:
  - GIT
tags:
  - 解决方案
  - GIT
abbrlink: e2e8d3d
date: 2025-02-06 14:57:00
---

``` powershell
# 比如合并分支时，test_file 冲突了，可以修改冲突并提交
<<<<<<< HEAD
当前分支内容
=======
合并进来的分支内容
>>>>>>> 分支名称

# 按照实际需求处理完冲突后，提交修改
$ git add test_file
$ git commit -m "合并test_file内容"
```
