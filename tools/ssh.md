---
title: SSH
date: 2018-03-29 21:51:47
tags: tech
---
1. No password
```
# copy local's .ssh/id_rsa.pub to remote servers' .ssh/authorized_keys
cat ~/.ssh/id_rsa.pub | ssh <user>@<hostname> 'cat >> .ssh/authorized_keys && echo "Key copied"'
# change the file privilege
chmod 644 authorized_keys
```
2. scp
3. -X


3. 保持当前进程运行退出
https://stackoverflow.com/questions/954302/how-to-make-a-programme-continue-to-run-after-log-out-from-ssh

ctrl+z
disown -h %1
bg 1
logout

fg
> 可以一并阅读一下
