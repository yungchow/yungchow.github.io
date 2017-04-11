## Git使用过程中遇到的一些问题

> 在github中新建的仓库，push本地仓库中的代码时报错:

`hint:Updates were rejected because the tip of your current branch is behind`

github上的仓库中自动生成了README.md文件，而本地中没有，导致版本冲突，需要先将远程仓库pull再push

于是:

`git pull origin master`

又报错:

`fatal:refusing to merge unrelated histories`

要把两个不同的项目合并，需要在添加`--allow-unrelated-histories`

于是:

`git pull origin master -- allow-unrelated-histories`

`git push -u origin master`



> 向本地仓库中add文件时报错:

`LF will be replace by CRLF fatal`

如下设置:

`git config core.autocrlf false`



> 从远程仓库中pull到本地

`You have not concluded your merge(MERGE_HEAD exist).Please,commit your changes before you can merge`

在本次pull之前的pull操作中没有自动合并，导致冲突

- 撤销merge操作并且重新pull
  - `git merge --abort`
  - `git reset --merge`
- 解决冲突
- add，commit这次合并
- `git pull`



