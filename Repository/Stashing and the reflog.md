# 暂存（Stashing）与引用日志（reflog）

在前面的章节，我们已经描述了把blobs放入到git仓库中的两个步骤：首先，在index中创建blobs，此时，这些blobs既没有父节点树，也没有能包含他的commit；然后，这些blobs被commit到Git仓库中，他们就像悬挂在被commit所包含的树上的叶子。本章节将会介绍另外两种把blob存储在git仓库中的方法。

第一种方法是`reflog`命令，它是一种元数据仓库，并以commit的形式记录了你对仓库进行的任何改变。这意味着，当你从index里创建了一棵树，并且使用commit把它存储在仓库里的同时，系统也会自动帮你把这条commit加入到reflog里，你可以使用下面的这条命令来浏览:

```
$ git reflog
5f1bc85...  HEAD@{0}: commit (initial): Initial commit
```

reflog的魅力在于，你的仓库中的其他改变，并不会影响reflog。这意味着我可以把上面这条commit操作从我的仓库中删除掉（使用 `reset`），然而这条commit仍然可以在30天内被reflog索引到，并且被reflog保护防止它被垃圾回收系统清理掉。这给了我一个月的时间去发现，是否真的想删除这条commit，并且在后悔删除时可以把它恢复回仓库中。

第二种方法是间接的使用你的工作树。我的意思是，当你已经改变了`foo.c`文件，但是你还没有把这些改变加入到index中时，Git此时可能并没有为已经改变的新文件创建blob，但是这些内容的改变却已经在文件系统中存在了，只是没有在Git仓库中存在罢了。此时，这个被改变的文件甚至都有了属于它的SHA1哈希码，尽管blob并没有真实的存在。你可以使用下面的命令看到它的SHA1哈希码：

```
$ git hash-object foo.c
<some hash id>
```

那么，你可以用这个特性，做什么事情呢？现在，假如你对你的工作树做了很多的修改，而且你今天也到了下班时间，保存你对工作树的修改是个好的习惯。

```
$ git stash
```

上面这条命令接收了你的文件夹中所有的内容——包括你的工作树以及此时index中的状态：先为当前的内容在git仓库中建立blobs，然后建立一棵树来包含这些blobs，同时还会生成两个commit来包含你此时的工作树以及index，并且记录你stash的时间。

使用存储命令是个好的习惯，虽然第二天你会使用`git apply`命令回到昨天存储之前的状态，但你可以通过存储命令获得每一天对工作树的修改记录。下面就是你第二天早上回来工作后会做的事情 。（WIP表示“Work in progress”）。

```
$ git stash list
stash@{0}: WIP on master: 5f1bc85...  Initial commit

$ git reflog show stash # same output, plus the stash commit's hash id 2add13e... stash@{0}: WIP on master: 5f1bc85... Initial commit

$ git stash apply
```

因为你存储的工作树保存在一个commit下，所以你可以在任何时候把它当做任意一个的分支使用它。这意味着你可以查看它的log，查看什么时候存储的它，并且检出到过去存储的工作树。

```
$ git stash list
stash@{0}: WIP on master: 73ab4c1...  Initial commit
...
stash@{32}: WIP on master: 5f1bc85...  Initial commit
$ git log stash@{32}  # when did I do it?
$ git show stash@{32}  # show me what I was working on
$ git checkout -b temp stash@{32}  # let’s see that old working tree!
```

最后一条命令格外的有用：它可以使我在一个月以前的没有提交过的工作树上工作。我从来没有把这些文件加入到index中，我只是在每天结束之前，使用了一个简单的叫做`stash`的命令把工作树和index存储起来（这也证明了你对工作树进行了修改），然后使用`stash apply`把存储的内容恢复回来。

如果你想清理你的存储列表——比如说，只保存最近30天的活动：不要使用`stash clear`！！！，请使用`reflog expire`命令。

```
$ git stash clear  # DON'T! You'll lose all that history
$ git reflog expire --expire=30.days refs/stash
<outputs the stash bundles that've been kept>
```

`stash`的魅力之处在于它可以让你用一种优雅的方式控制你的工作进程，换句话说，它可以记录你工作阶段的每一天。如果你愿意，你甚至可以定期使用`stash`，例如下面这个快照脚本：

```
$ cat <<EOF > /usr/local/bin/git-snapshot
#!/bin/sh
git stash && git stash apply
EOF
$ chmod +x $_
$ git snapshot
```

你可以使用`cron`定时任务，每小时执行上面的脚本一次，并且每周或者每个月使用`reflog expire`清理一部分存储。