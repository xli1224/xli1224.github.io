---
layout: post
title: 为 git 库配置多个 remote
tags:
- Git
---

当存在多个远端库需要同步时，可以为本地库配置多个 remote，每次 push 或者 pull 的时候指定来源。

比如说有两个远端库 A 和 B。从任意一个库 clone 至本地后运行
`git remote -v`

会发现当前的 push 和 fetch 都是关联的 A，并且名字叫做 origin。
```
origin	ssh://xiang@192.168.1.91:50022/opt/gitrepo/withholdings/WithholdingBill (fetch)
origin	ssh://xiang@192.168.1.91:50022/opt/gitrepo/withholdings/WithholdingBill (push)
```

为它添加另一个库，是文件系统的另一个位置。
```
git remote add local /tmp/withholdingBill.git
```

再次查看，就发现有两个 remote，都可以推送和拉取。
```
$ git remote -v
local	/tmp/withholdingBill.git (fetch)
local	/tmp/withholdingBill.git (push)
origin	ssh://xiang@192.168.1.91:50022/opt/gitrepo/withholdings/WithholdingBill (fetch)
origin	ssh://xiang@192.168.1.91:50022/opt/gitrepo/withholdings/WithholdingBill (push)
```

推送和拉取的时候指定用哪个 remote 即可。

并且这个方式有个好处就是从 A 库拉取下来的更新，可以直接推送到 B 库去。

![image](/assets/images/81F1097A-CA7C-49F6-ABE4-877DF26203B9.png)
