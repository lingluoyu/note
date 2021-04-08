### Git 配置

#### 配置信息

```bash
#查看所有配置信息
git config --list
#查看系统配置
git config --system --list
#查看全局配置
git config --global --list
```

#### 设置用户名和邮箱

```bash
git config --global user.name "lingluoyu"
git config --global user.email "lingluoyu@qq.com"
```

#### 配置忽略文件

在工作目录主目录下建立 .gitignore 文件

```bash
*.txt #忽略所有.txt结尾的文件
!a.txt  #a.txt除外
temp/ #忽略temp目录下的文件
```

#### SSH 免密登录

```bash
ssh-keygen -t rsa -C "lingluoyu@qq.com"
```

连续三次回车后在.ssh目录下会生成一个id_rsa和id_rsa.pub，把id_rsa.pub中的字符串保存到 github 设置中的ssh公钥中，即可免密提交下载代码

### Git 工作原理

Git 分为四个区域：

- **工作目录**（Working Directory）：存放项目代码的目录，电脑中可以看到的目录
- **暂存区**（Stage）：临时存放改动，一般存放在 **.git** 目录下的 index 文件（.git/index）中，所以我们把暂存区有时也叫作索引（index）
- **资源库**（Repsitory/Git Directory）：工作区有一个隐藏目录 **.git**，这个不算工作区，而是 Git 的版本库
- **远程仓库**（Remote Directory）：代码托管的平台

**工作目录-->git add files-->暂存区-->git commit-->资源库-->git push-->远程仓库**

![img](https://gitee.com/LoopSup/image/raw/master/img/git-01.jpeg)

- 图中左侧为工作区，右侧为版本库。在版本库中标记为 "index" 的区域是暂存区（stage/index），标记为 "master" 的是 master 分支所代表的目录树。
- 图中我们可以看出此时 "HEAD" 实际是指向 master 分支的一个"游标"。所以图示的命令中出现 HEAD 的地方可以用 master 来替换。
- 图中的 objects 标识的区域为 Git 的对象库，实际位于 ".git/objects" 目录下，里面包含了创建的各种对象及内容。
- 当对工作区修改（或新增）的文件执行 **git add** 命令时，暂存区的目录树被更新，同时工作区修改（或新增）的文件内容被写入到对象库中的一个新的对象中，而该对象的ID被记录在暂存区的文件索引中。
- 当执行提交操作（git commit）时，暂存区的目录树写到版本库（对象库）中，master 分支会做相应的更新。即 master 指向的目录树就是提交时暂存区的目录树。
- 当执行 **git reset HEAD** 命令时，暂存区的目录树会被重写，被 master 分支指向的目录树所替换，但是工作区不受影响。
- 当执行 **git rm --cached <file>** 命令时，会直接从暂存区删除文件，工作区则不做出改变。
- 当执行 **git checkout .** 或者 **git checkout -- <file>** 命令时，会用暂存区全部或指定的文件替换工作区的文件。这个操作很危险，会清除工作区中未添加到暂存区的改动。
- 当执行 **git checkout HEAD .** 或者 **git checkout HEAD <file>** 命令时，会用 HEAD 指向的 master 分支中的全部或者部分文件替换暂存区和以及工作区中的文件。这个命令也是极具危险性的，因为不但会清除工作区中未提交的改动，也会清除暂存区中未提交的改动。

### 使用

#### git init

Git 使用 **git init** 命令来初始化一个 Git 仓库，Git 的很多命令都需要在 Git 的仓库中运行，所以 **git init** 是使用 Git 的第一个命令。

在执行完成 **git init** 命令后，Git 仓库会生成一个 .git 目录，该目录包含了资源的所有元数据，其他的项目目录保持不变。

```bash
# 使用当前目录作为Git仓库
git init
# 指定目录作为Git仓库
git init newrepo
```

#### 常用命令

##### 创建仓库

| 命令        | 说明                                   |
| :---------- | :------------------------------------- |
| `git init`  | 初始化仓库                             |
| `git clone` | 拷贝一份远程仓库，也就是下载一个项目。 |

##### 提交与修改

| 命令         | 说明                                     |
| :----------- | :--------------------------------------- |
| `git add`    | 添加文件到仓库                           |
| `git status` | 查看仓库当前的状态，显示有变更的文件。   |
| `git diff`   | 比较文件的不同，即暂存区和工作区的差异。 |
| `git commit` | 提交暂存区到本地仓库。                   |
| `git reset`  | 回退版本。                               |
| `git rm`     | 删除工作区文件。                         |
| `git mv`     | 移动或重命名工作区文件。                 |

##### 提交日志

| 命令               | 说明                                 |
| :----------------- | :----------------------------------- |
| `git log`          | 查看历史提交记录                     |
| `git blame <file>` | 以列表形式查看指定文件的历史修改记录 |

##### 远程操作

| 命令         | 说明               |
| :----------- | :----------------- |
| `git remote` | 远程仓库操作       |
| `git fetch`  | 从远程获取代码库   |
| `git pull`   | 下载远程代码并合并 |
| `git push`   | 上传远程代码并合并 |

##### 分支管理

| 命令                                                         | 说明                               |
| :----------------------------------------------------------- | :--------------------------------- |
| `git branch`                                                 | 列出所有分支                       |
| `git branch -r`                                              | 列出所有远程分支                   |
| `git branch [branch-name]`                                   | 新建一个分支，但依然停留在当前分支 |
| `git checkout -b [branch]`                                   | 新建一个分支，并切换到该分支       |
| `git merge [branch]`                                         | 合并指定分支到当前分支             |
| `git branch -d [branch-name]`                                | 删除分支                           |
| `git push origin --delete [branch-name]` `git branch -dr [remote/branch]` | 删除远程分支                       |

```bash
#列出所有分支
git branch

#列出所有远程分支
git branch -r

#新建一个分支，但依然停留在当前分支
git branch [branch-name]

#新建一个分支，并切换到该分支
git checkout -b [branch]

#合并指定分支到当前分支
git merge [branch]

#删除分支
git branch -d [branch-name]

#删除远程分支
git push origin --delete [branch-name]
git branch -dr [remote/branch]
```

##### 查看提交历史

| 命令               | 说明                                 |
| :----------------- | :----------------------------------- |
| `git log`          | 查看历史提交记录                     |
| `git blame <file>` | 以列表形式查看指定文件的历史修改记录 |

##### 标签

达到一个重要的阶段，并希望永远记住那个特别的提交快照，你可以使用 git tag 给它打上标签。

| 命令                             | 说明                 |
| :------------------------------- | :------------------- |
| `git tag -a <tagName>`           | 创建一个带注解的标签 |
| `git tag -a <tagName> [logId]`   | 追加标签             |
| `git tag -a <tagname> -m "标签"` | 指定标签信息         |