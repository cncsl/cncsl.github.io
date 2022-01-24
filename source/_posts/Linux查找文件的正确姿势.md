---
title: Linux查找文件的正确姿势
date: 2021-02-04 21:40:38
categories: Linux
tags: [Linux]
---

Linux 系统中查找文件的命令有 `which`、`whereis`、`locate` 和 `find` 等，本文对这四条命令进行简单的介绍、列举了一些简单的使用方式。

<!--more-->

# which

**在 PATH 变量中定义的全部路径中查找可执行文件或脚本**。

`which` 命令有两个重要参数：

- `-all, -a` 默认情况下，`which` 命令会在匹配到第一个结果后结束运行，添加该参数可以让其搜索所有路径。

- `-read-alias, -i` 将输入视为别名搜索。Linux 系统中通常会使用 alias 设置诸多别名来简写命令，例如 Centos 中的 `ll` 实际是 `ls -l` ，而 `which` 是 `alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde`。
  
  ```shell
  # Centos
  # 以绝对路径调用 which，这样就不会受到 Centos 默认的几个参数影响
  # 返回结果说明找不到 ll 命令
  $ /usr/bin/which ll
  /usr/bin/which: no ll in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin)
  
  # 直接输入 which 时实际效果为带有“默认参数”的
  # 返回结果说明 ll 是 ls -l 的别名，
  $ which ll
  alias ll='ls -l --color=auto'
          /usr/bin/ls
  ```
  
  `which ll` 相当于 `alias | /usr/bin/which --tty-only --read-alias --show-dot --show-tilde ll`，返回结果第一行是 alias 输出的 `ll` 别名设置情况，第二行则是 `ls` 的实际位置。

`which` 的其他几个参数如下：

- `--tty-only`：尽在终端调用的情况下附带右侧添加的参数，其他情况下不接收右侧其他参数（此处的参数值 `--show-dot`、`--show-tilde` 此类，输入的待查询命令仍然会接收），通过这个命令可以保证 Shell 脚本中的 `which` 命令正确执行。
- `--show-dot`：输出以 "." 符号开头的目录。Linux 中 "." 符号开头的目录是约定的隐藏文件夹，没有该参数时会忽略这些目录。
- `--show-tilde`：将用户家目录替换成 "~" 符号输出。Linux 中 "~" 符号是登录用户家目录的缩写，如果登录用户名为 cncsl，则 "~" 指 "/home/cncsl" 目录。当使用 root 账号登录时该参数无效。

# whereis

**查找指定命令的可执行文件、源代码和手册的位置。**

```shell
$ whereis vim
vim: /usr/bin/vim /usr/share/vim /usr/share/man/man1/vim.1.gz
```

可以看出，`vim` 的可执行程序位于 `/usr/bin/vim`，手册位于 `/usr/share/vim` 和 `/share/man/man1/vim.1.gz` 目录。

- `-b`、`-m` 和 `-s` 分别用于指定仅查询可执行文件、手册和源代码。

- `-B`、`-M`和 `-S` 命令用于指定查询路径。

- `-u` 参数的描述直译为 *仅查询有异常情况的命令*。所谓的异常情况是指，某个命令的相关类型文件不止恰好一份（一份都没有或多于一份）。例如：
  
  - `ls` 命令具有两份手册：
    
    ```shell
    $ whereis -m -u ls
    ls: /usr/share/man/man1/ls.1.gz /usr/share/man/man1p/ls.1p.gz
    ```
  
  - Linux 系统中有很多个与 python 相关的可执行文件：
    
    ```shell
    $ whereis -b -u python
    python: /usr/bin/python /usr/bin/python2.7 /usr/lib/python2.7 /usr/lib64/python2.7 /etc/python /usr/include/python2.7
    ```

# locate

**在文档和目录名称的数据库中查找指定文件。**Linux 系统会定期自动扫描磁盘来维护一个记录磁盘数据的数据库，而 `locate` 命令使用的数据库是 */var/lib/mlocate/mlocate.db*。

```shell
$ ls -hl /var/lib/mlocate/mlocate.db
-rw-r-----. 1 root slocate 2.7M Feb  4 03:42 /var/lib/mlocate/mlocate.db
```

可以看出当前 *mlocate.db* 文件共记录了 2.7M 的数据。

- `--count, -c` ：不输出具体的文件路径信息，仅输出查询到的数量。
- `--ignore-case, -i`：查询时忽略大小写
- `--limit, -l, -n LIMIT`：限定输出的文件数量为 LIMIT
- `--regexp,-r REGEXP`：使用 REGEXP 指定的正则表达式匹配。

```shell
# 统计有多少PNG格式的图像文件
$ locate -c png

# 统计有多少 readme 文件（根据编写者的习惯，readme 文件可能名为 README、ReadMe等）
$ locate -c -i readme

# 输出十个 .gz 归档文件的路径
$ locate -l 10 *.gz

# 查看 tomcat 2021年1月的日志
$ locate -r tomcat.2021-01-[0-3][0-9].log
```

由于 `locate` 命令是从数据库查找文件，新创建的文件可能由于未被记录到数据库中而无法查询到，这种时候需要使用 `updatedb` 命令手动更新数据库。

# find

**在一个目录层级中查找文件。**

`find` 命令功能强大，可根据多种条件查询文件，随后进行自定义的操作，格式如下：

```shell
find [path...] [expression]
```

- 查询当前目录下所有的 markdown 文档：
  
  ```shell
  $ find . -name "*.log"
  ```

- 查询用户视频文件夹中大于 100M 的文件：
  
  ```shell
  $ find ~/Videos/ -size +100M
  ```

- 查询用户音乐文件夹中过去七天访问过的文件：
  
  ```shell
  $ find ~/Music/ -atime -7
  ```

- 查询系统中、三个月之前创建的、一个月之内没有访问过、大于 30M 的日志文件，并删除：
  
  ```shell
  find / -ctime +90 -atime +30 -size +1M -name "*.log" -delete
  ```

`find` 会实际的扫描磁盘，所以速度会明显小于前三个。