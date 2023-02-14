# Zookeeper ACL权限控制讲解

`Zookeeper`使用`ACL（Access Control List）`来控制对`Znode`的访问

ACL仅对特定的`Znode`有效，也就是说它不是递归的，`/a`的ACL权限对`/a/b`无效。

# Zookeeper权限种类

`Zookeeper`中的权限有五种

* CREATE：创建子节点
* READ：读取一个节点的值，或者列出某一个节点的子节点。
* WRITE：给节点设置值
* DELETE：删除节点
* ADMIN：可以设置权限

每种权限的首字母（小写）

# Zookeeper ACL授权方案（Builtin ACL Schemes）

`Zookeeper`内置了5中内置的ACL授权方案

* world：只有一个用户：anyone，代表登录zokeeper所有人（默认）
* ip：对客户端使用IP地址认证
* auth：使用已添加认证的用户认证
* digest：使用“用户名:密码”方式认证
* x509：用客户端 X500 主体作为 ACL ID 标识。ACL 表达式是客户端的确切 X500 主体名称。使用安全端口时，将自动对客户端进行身份验证，并设置 x509 方案的身份验证信息。

# ACL基本操作

接下来使用命令行来实际操作下。

通过`zkCli.sh`连接到`Zookeeper`
```shell
$ ./zkCli.sh -server localhost:2181
Connecting to localhost:2181
------省略N行------
WATCHER::

WatchedEvent state:SyncConnected type:None path:null
[zk: localhost:2181(CONNECTED) 0]

```
## 默认权限
默认情况下，一个`Znode`被创建默认赋予`world`权限。
```shell
[zk: localhost:2181(CONNECTED) 2] create /world1
Created /world1
[zk: localhost:2181(CONNECTED) 3] getAcl /world1
'world,'anyone
: cdrwa

```
上面日志中的`world`代表内置的Schema，anyone是`world`的唯一ID。`rdraw`则代表权限种类，也就是说默认`world`具有`CREATE`，`READ`，`WRITE`，`DELETE`，`ADMIN`五种权限。

## 获取权限

上面已经使用过了，通过`getAcl`指令获取权限。
```shell
[zk: localhost:2181(CONNECTED) 3] getAcl /world1
'world,'anyone
: cdrwa
```

## 设置权限
设置权限可以有两种方法，一种是使用`create [Node] [ACL]`来设置，另外一种是通过`setACL  [Node] [ACL]`方法来设置。

> 这里要特别说明[ACL]的格式是`Schema:Schema标识:[cdrwa]`。

### world

world是默认权限

```shell
[zk: localhost:2181(CONNECTED) 4] create /a1 
Created /a1
```
### IP
IP模式其实类似白名单的模式，多个IP需要使用逗号分隔。

通过上面代码我已经创建了一个默认`world`的节点`/a1`, 接下来我修改该节点的权限为IP的方案。
```shell
[zk: localhost:2181(CONNECTED) 0] setAcl /a1 ip:127.0.0.1:cdrwa

```


因为我设置了`ip:127.0.0.1:cdrwa`，所以当前服务器具有`cdrwa`操作权限。
```shell
[zk: localhost:2181(CONNECTED) 4] getAcl /a1
'ip,'127.0.0.1
: cdrwa
[zk: localhost:2181(CONNECTED) 5] get /a1
null

```
可以看到本机是可以正常操作的。

接下来我换一个服务器来观察下能否正常操作`/a1`节点：

```shell
[zk: 10.112.33.229(CONNECTED) 0] get /a1
Insufficient permission : /a1
```
可以看到提示`Insufficient permission`权限不足。

### auth
auth 是通过明文用户名密码来认证的,首先要使用`addauth`指令，再使用`create`或者`setAcl`设置权限。
```shell
[zk: localhost:2181(CONNECTED) 6] addauth digest user:password
[zk: localhost:2181(CONNECTED) 7] create /t1 data auth:user:password:cdwra
Created /t1
[zk: localhost:2181(CONNECTED) 8] getAcl /t1
'digest,'user:tpUq/4Pn5A64fVZyQ0gOJ8ZWqkY=
: cdrwa
[zk: localhost:2181(CONNECTED) 9]
```
可以看到能够正常获取到数据，因为当前会话就是用户`user`。

重新开启一个客户端，不执行` addauth digest user:password`(也就是是说当前会话并未指定用户`user`信息)：
```shell
[zk: localhost:2181(CONNECTED) 0] ls /
[t1, zookeeper]
[zk: localhost:2181(CONNECTED) 1] get /t1
Insufficient permission : /t1
```
提示权限不足。

这就说明了`auth`模式下，用户的作用。

### digest 
digest 类似与auth，只不过他是通过密文授权。

首先要提供密码的密文，生成方式有多种，我通过shell指令+openssl来生成。
```shell
$ echo -n myuser:mypassword | openssl dgst -binary -sha1 | openssl base64
S+PxW8CqJU0w30A4/j6ewczsru0=
```
拿到密文后，就可以授权了
```shell
[zk: localhost:2181(CONNECTED) 9] create /t2 t2_value digest:myuser:S+PxW8CqJU0w30A4/j6ewczsru0=:cdrwa
Created /t2
```
授权完成后，查询`/t2`的Acl信息以及值。
```shell
[zk: localhost:2181(CONNECTED) 2] getAcl /t2
Insufficient permission : /t2
[zk: localhost:2181(CONNECTED) 3] get /t2
Insufficient permission : /t2
```
提示没有权限，没有问题，以为当前会话并没有设置用户。

通过`addauth`设置后，重新查询
```shell
[zk: localhost:2181(CONNECTED) 0] addauth digest myuser:mypassword
[zk: localhost:2181(CONNECTED) 1] getAcl /t2
'digest,'myuser:S+PxW8CqJU0w30A4/j6ewczsru0=
: cdrwa
[zk: localhost:2181(CONNECTED) 2] get /t2
t2_value
```
没问题可以正常获取。

### x509 





