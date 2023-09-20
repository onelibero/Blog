此处以centos7.6版本的服务器进行操作

## 1.下载rpm

### （1）[erlang](https://packagecloud.io/rabbitmq/erlang/packages/el/7/erlang-23.2.7-2.el7.x86_64.rpm)

### （2）[RabbitMQ](https://github.com/rabbitmq/rabbitmq-server/releases/download/v3.8.14/rabbitmq-server-3.8.14-1.el7.noarch.rpm)

## 2.进入shell面板

```shell
#连接到服务器后，进入默认root目录创建文件夹
mkdir rabbitmq
#将下载的rpm压缩包传到该目录下
#进行解压
rpm -Uvh *************.rpm（erlang）
rpm -Uvh *************.rpm（rabbitmq）

#安装erlang
yum install -y erlang
#查看版本检测是否安装成功
erl -v


#安装rabbitmq
#先安装socat插件
yum install -y socat
yum install -y rabbitmq-server
# 启动rabbitmq
systemctl start rabbitmq-server
# 查看rabbitmq状态，出现active状态表示成功
systemctl status rabbitmq-server


# 设置rabbitmq服务开机自启动
systemctl enable rabbitmq-server
# 关闭rabbitmq服务
systemctl stop rabbitmq-server
# 重启rabbitmq服务
systemctl restart rabbitmq-server

# 打开RabbitMQWeb管理界面插件
rabbitmq-plugins enable rabbitmq_management
```

打开服务器公网ip:15672即可进入管理页面

默认guest（账号密码）本机的

所以创建角色

- `administrator`：可以登录控制台、查看所有信息、并对rabbitmq进行管理
- `monToring`：监控者；登录控制台，查看所有信息
- `policymaker`：策略制定者；登录控制台指定策略
- `managment`：普通管理员；登录控制

```shell
# 添加用户
rabbitmqctl add_user 用户名 密码

# 设置用户角色,分配操作权限
rabbitmqctl set_user_tags 用户名 角色

# 为用户添加资源权限(授予访问虚拟机根节点的所有权限)
rabbitmqctl set_permissions -p / 用户名 ".*" ".*" ".*"

# 修改密码
rabbitmqctl change_ password 用户名 新密码

# 删除用户
rabbitmqctl delete_user 用户名

# 查看用户清单
rabbitmqctl list_users
```