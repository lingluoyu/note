git bash 客户端第一次查看文件时，无法正确显示中文的文件名，输入以下命令即可解决

```bash
git config --global core.quotepath false          # 显示 status 编码
git config --global gui.encoding utf-8            # 图形界面编码
git config --global i18n.commit.encoding utf-8    # 提交信息编码
git config --global i18n.logoutputencoding utf-8    # 输出 log 编码
export LESSCHARSET=utf-8
#最后一条命令是因为 git log 默认使用 less 分页，所以需要 bash 对 less 命令进行 utf-8 编码
```

