# Installing on Debian / Ubuntu

+ 添加 ``` /etc/apt/preferences.d/erlang ```

```
# /etc/apt/preferences.d/erlang
Package: erlang*
Pin: version 1:19.3-1
Pin-Priority: 1000

Package: esl-erlang
Pin: version 1:19.3.6
Pin-Priority: 1000
```

+ 验证包

```
sudo apt-cache policy
```

+ 安装

```
sudo apt install rabbitmq-server -y
```

+ 查看rabbitmq-server 服务

```
service rabbitmq-server status
```