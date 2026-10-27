切换root

su -

输入root账号密码

连接超时问题：

修改服务端的配置：

```bash
vim /etc/ssh/sshd_config

# server每隔60秒发送一次请求给client，然后client响应，从而保持连接
ClientAliveInterval 60

＃server发出请求后，客户端没有响应得次数达到3，就自动断开连接，正常情况下，client不会不响应        
ClientAliveCountMax 10

# 重载配置
sudo systemctl reload ssh

```

修改客户端的配置：

```bash

vim /etc/ssh/ssh_config
ServerAliveInterval 60
ServerAliveCountMax 3
```

