# git基操指南

**git init**

**git clone**                  --不加参数克隆后本地的HEAD头会指向默认分支，建议使用

**git clone -b {远程分支名}** ，这会使本地HEAD头指向你想要的分支。注意，这并没有在本地创建新分支！

**cd {repository}**      --很重要，不进去接下来啥命令都没用

**git remote add origin ssh://git@git.idata.wk:8022/datainfrastructure/regional/backend.git**

​                                  --设置远程仓库

**git remote -v**         --验证远程仓库是否已成功添加

**git branch -a**          --查看所有本地仓库的远程分支

**git checkout {远程分支名}**

​                                  --这个仅用于clone没带参数时，需要手动切换本地HEAD头指向。注意，这并没有在本地创建新分支！

**git checkout -b {你的新本地分支名} {origin/远程分支名}**

​                                   --基于一个远程分支创建一个新的本地分支

**git branch -vv**         --展示本地分支和远程分支对应关系，可验证执行上一行命令后新本地分支是否正确地跟踪了远程分支

 