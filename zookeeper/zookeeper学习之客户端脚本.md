# 一、客户端脚本

## 1、客户端启动脚本

```shell
$ sh zkCli.sh
```

上边脚本是启动一个本地的zk服务器，如果想启动指定服务器地址，则可以执行如下命令：

```shell
$ sh zkCli.sh -server ip:port
```

## 2、创建节点

可以使用`create`命令，创建一个zookeeper节点，如下

```shell
$ create [-s] [-e] path data acl
```

其中，-s和-e分别指定节点特性：顺序或临时节点。默认情况下，创建的是持久节点，`create`命令的最后一个参数是acl，它是用来进行权限控制的，缺省情况下，不做任何权限控制

## 3、读取

与读取相关的命令包括`ls`命令和`get`命令

### 1）ls

使用`ls`命令，可以列出zookeeper指定节点下的所有**第一级**子节点，用法如下：

```shell
ls path [watch]
```

其中，path标识的是指定数据节点的节点路径

### 2）get

使用`get`命令，可以获取到zookeeper指定节点的数据内容和属性信息，用法如下：

```shell
get path [watch]
```

执行该命令后，除了获取到数据外，还获取到一些关于节点的信息：

a、创建该节点的事物ID：cZxid

b、最后一次更新该节点的事物ID：mZxid

c、最后一次更新该节点的时间：mtime

e、第一次更新该节点的时间：ctime

### 3）更新

使用`set`命令，可以更新指定节点的数据内容，用法如下：

```shell
set path data [version]
```

其中，data就是要更新的内容，version参数指的是本次更新操作是基于ZNode的哪一个数据版本进行的

### 4）删除

使用delete命令，可以删除zookeeper上的指定节点，用法如下：

```shell
delete path [version]
```

此处的version参数和`set`命令中的version参数的作用是一致的，但是要注意的是，**想要删除某一个指定节点，那么该节点必须没有子节点存在**

# 二、zookeeper的java客户端

zookeeper客户端和服务端会话的建立是一个异步的过程，也就是说在程序中，构造方法会在处理完客户端初始化工作后立即返回，在大多数情况下，此时并没有真正建立好一个可用的会话，在会话的生命周期中处于“CONNECTING”状态。当该会话真正创建完毕后，zookeeper服务端会向会话对应的客户端发送一个事件通知，以告知客户端，客户端只有在获取这个通知之后，才算真正建立了会话

## 1、创建

zookeeper不支持递归创建，即无法在父节点不存在的情况下创建一个子节点。另外，如果一个节点已经存在了，那么创建同名的节点的时候，会抛出异常

目前，zookeeper的节点内容只支持字节数组类型，也就是说，zookeeper不负责为节点内容进行序列化，开发人员需要自己使用序列化工具将节点内容进行序列化和反序列化

## 2、读取数据

zookeeper的读取支持在客户端获取到指定节点的子节点列表后，再订阅这个子节点列表的变化通知，这个功能可以通过注册Watcher的手段来实现。当有子节点被添加或者删除时，服务端就会向客户端发送一个NodeChildrenChanged(EventType.NodeChildernChanged)类型的事件通知。需要注意的是，在服务端发送给客户端的事件通知中，是不包含最近的节点列表的，客户端必须主动重新进行获取。

## 3、更新操作

zookeeper中的更新操作都会附带一个版本号，这个版本号用于指定节点的数据版本，表明本次更新操作是针对指定数据版本进行的。在zookeeper中，数据版本都是从0开始计数的，如果在更新节点的时候传入了一个-1，因为-1并不是一个合法的数据，它在zookeeper中仅仅是作为一个标识符，这个时候就是告诉zookeeper服务器，客户端需要基于数据的最新版本进行更新操作。**如果对zookeeper数据节点的更新操作没有原子性要求，那么就可以使用-1**

## 4、检测节点是否存在

可以使用exists来检测指定节点是否存在，并注册一个监听器来监听该节点的创建、删除、更新操作

有以下几点需要注意：

1）无论指定节点是否存在，通过调用exists接口都可以注册Watcher

2）exists接口中注册的Watcher，能够对节点创建、节点删除和节点数据更新事件进行监听

3）对于指定节点的子节点的各种变化，都不会通知客户端

## 5、权限控制

为了避免存储在zookeeper服务器上的数据被其它进程干扰或人为操作修改，需要对zookeeper上的数据访问进行权限控制。zookeeper提供了ACL的权限控制机制，简单的讲，就是通过设置zookeeper服务器上数据节点的ACL，来控制客户端对该数据节点的访问权限：如果一个客户端符合该ACL控制，那么就可以对其进行访问，否则将无法访问。zookeeper提供了以下几种ACL控制策略：

- world
- auth
- digest
- ip
- super

删除操作中，对于添加了权限信息的子节点，其作用范围是子节点，也就是说，当我们对一个数据节点添加权限信息后，依然可以自由地删除这个节点，但是对于这个节点的子节点，就必须使用相应的权限信息才能够删除掉它

**注：zookeeper规定，所有的非叶子节点必须为持久节点**

