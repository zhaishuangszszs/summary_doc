* `git push` 
第一次
`git push --set-upstream <remote> <branchname>` 
后面直接git push就好了，**目的**：当前分支与远程分支关联。

* `git branch`
`git branch`或者`git branch -v`查看分支
`git branch -d/D <branchname>`删除分支
* `git rebase`
[here](https://git-scm.com/docs/git-rebase/zh_HANS-CN) 
[here](http://jartto.wang/2018/12/11/git-rebase/)
`git rebase -i HEAD~n`合并最近n次提交

* `git stash`

* `git pull` 
`git pull <远程主机名> <远程分支名>`
`git pull <远程主机名> <远程分支名>:<本地分支名>`是**错误的，不能指定本地分支名字**

* `git merge`时显示`Please enter a commit message to explain why this merge is necessary, especially if it merges an upd`要`CTRL + []`执行命令



### doc
git log				(空格向下翻页，b向上翻页)。
git reset --hard HEAD~n		（n回退版本数）
git restore <file>		以下命令可以将指定文件 <file> 恢复到最新的提交状态，丢弃所有未提交的更改




删除缓冲区中的文件：
1.git rm --cached "文件路径"，不删除物理文件，仅将该文件从缓存中删除；
1.git rm --f "文件路径"，不仅将该文件从缓存中删除，还会将物理文件删除（不会回收到垃圾桶）；

如果一个文件已经add到暂存区，还没有 commit，此时如果不想要这个文件了，有两种方法：
1.用版本库内容清空暂存区，git reset HEAD 回退到当前版本（在Git中，用HEAD表示当前版本，上一个版本就是HEAD^，上上一个版本就是HEAD^^，当然往上100个版本写100个^比较容易数不过来，所以写成HEAD~100）；
2.只把特定文件从暂存区删除，git rm --cached xxx；


// 跳转到指定版本
git reset --hard  fa4bf08fed85fc0ca5acde22464e68c6f8cfc8f2


git push <远程主机名> <本地分支名>:<远程分支名>


//创建临时分支
git pull <远程主机名> <远程分支名>:<本地分支名>
//不创建临时分支，只获取更新
git fetch <远程主机名> <远程分支名>:<本地分支名>


## 错误提示
* `'main/' does not have a commit checked out`
一般是子目录下有`.git`文件，子目录需要`add&commit`


## git 提交流程
* `git checkout main`
* `git pull upstream main`
* `git push lakesoul_zs main`
* `git checkout zs`
* `git merge main`
* `git commit -s -m "update"`
* `git push lakesoul_zs zs`