---
title: rbash bypass
date: 2019-05-19 01:38
---

rbash 是 Linux 下的受限 bash, 可以提高安全性, 或是用来限制用户的行为以免用户误操作. 
rbash限制以下行为的执行:
- cd 切换目录
- 含有斜杠 `/` 的命令, 譬如 `/bin/sh`
- 设置 PATH ENV 等环境变量
- 使用 `>` `<` 进行重定向
- binary 的运行. 通常 root 用户会手动创建 `/bin/binary_file -> /home/rbash_user/bin/binary_file` 的软链接, 限制性地提供部分 binary_file 给 rbash_user 使用
在 bash 下 `echo $SHELL`, 可以获取当前环境是否是 rbash.

## 配置 rbash
参考这篇 https://www.ostechnix.com/how-to-limit-users-access-to-the-linux-system/

主要步骤:
```sh
#useradd rbash_test --create-home -s /bin/rbash
#passwd rbash_test
#cd /home/rbash_test 
#mkdir bin
#ln -s /bin/ls $(pwd)/bin/ls
#ln -s /bin/xxx_bin $(pwd)/bin/xxx_bin
#echo "export PATH=$HOME/bin" > .bash_profile
```
这样就创建了一个用户名为 rbash_test 的受限账户, 并且账户只被允许 $HOME/bin 下的内容.

## rbash 绕过
绕过 rbash , 主要取决于 rbash user 所被允许执行的命令. 通常是利用软件执行外部命令来绕过 rbash. 常见情形的有 vim set variable, python 调用 os.system 等.
实际上就是换一个形式调出 bash. 跳脱出 rbash 的方法和 rbash 本身的限制并没有太大关系....
- scp 绕过

    rbash 账户的 bin 目录可能由 root 用户创建的, 这时候由于权限问题, rbash 用户无法写入 bin 目录, 也就意味着没有办法用 scp  把 bash 复制到 rbash 用户的 bin 目录下使用. 但 scp 可以查看敏感文件内容
`scp -F /etc/passwd a b:`  

- 利用 vim 绕过

    如果允许使用 vim, 那么可以透过 set variable 的方式来启动一个拥有完全权限的 bash
打开vim, 然后 
    ```
:set shell=/bin/bash
:shell
    ```
    就能启动 bash 了, 不过进入 bash 后, 因为 $PATH 还是原来的 rbash user 的 $PATH, 执行一些命令可能会报错找不到路径, 可以自己 set $PATH 或是用绝对路径运行.

- python 绕过

    直接进入 python , `import os` `os.system("/bin/bash")` 就行. 
