# RPC框架

## 提问

1.怎么实现的同步？你的调用为什么是同步的？ 

答： Netty是同步非阻塞的，我在项目中使用的netty进行网络传输。



2.客户端发请求的时候，封装请求之后通过tcp发送到客户端的吗？

答：是的

 

3.客户端和服务端都用的tcp？ server那边怎么处理的请求

答：都用的tcp， server那边先反序列化获得请求类，请求类RPCRequest里面包含需要调用的方法和参数，执行对应方法后会把结果封装成RPCResponse类返回给客户端



4.是在netty的线程里面直接调用了业务代码吗？

答：不是，netty传输的是序列化后的字节码。需要在服务端桩反序列化后执行业务代码。



## 实现过程中的自问自答

1. JDK的动态代理 ——> method.invoke()体现在何处？

   答：服务端

2. 传输协议如何设计（数据包的设计）

   ![image-20210319191613569](C:\Users\95845\AppData\Roaming\Typora\typora-user-images\image-20210319191613569.png)

   + magic code魔数：通常是4个字节。这个魔数主要是为了筛选来到服务端的数据包，有这个魔数之后，服务端首先取出前面四个字节进行比对，能够在第一时间识别出这个数据包并非是遵循自定义协议的，也就是无效数据包，为了安全考虑可以直接关闭连接节省资源。
   + full length：消息长度，运行时计算出来
   + version =（byte） 1
   + **MessageType:**  REQUEST_TYPE = 1, RESPONSE_TYPE = 2, HEARTBEAT_REQUEST_TYPE = 3, HEARTBEAT_RESPONSE_TYPE = 4
   + **codec: **序列化类型：kryo或者protostuff
   + **Compress： **压缩方式   枚举类GZIP((byte)  0x01, "gzip")

3. 从哪里开始调用的动态代理对象？

   答：1）先通过@RpcScan注解，调用CustomScannerRegister类

   ​		2）创建了SpringBeanPostProcessor类实现了BeanPostProcessor接口，目的是在IOC容器生成bean对象前后，将该client端需调用的bean对象重新设置成动态代理类proxy

4. 在整个框架中有使用spring容器吗？仅仅在SpringBeanPostProcessor中看到了@Component注解，不清楚NettyRpcClient类是何时初始化的？

5. netty的心跳机制

   心跳机制的工作原理是: 在 client 与 server 之间在一定时间内没有数据交互时, 即处于 idle 状态时, 客户端或服务器就会发送一个特殊的数据包给对方, 当接收方收到这个数据报文后, 也立即发送一个特殊的数据报文, 回应发送方, 此即一个 PING-PONG 交互。所以, 当某一端收到心跳消息后, 就知道了对方仍然在线, 这就确保 TCP 连接的有效性.

   TCP 实际上自带的就有长连接选项，本身是也有心跳包机制，也就是 TCP 的选项：`SO_KEEPALIVE`。 但是，TCP 协议层面的长连接灵活性不够。所以，一般情况下我们都是在应用层协议上实现自定义心跳机制的，也就是在 Netty 层面通过编码实现。通过 Netty 实现心跳机制的话，核心类是 `IdleStateHandler` 。
   
6. Netty如何解决粘包问题（拆包策略）

   > Netty对解决粘包和拆包的方案做了抽象，提供了一些解码器（Decoder）来解决粘包和拆包的问题。如：
   >
   > - LineBasedFrameDecoder：以行为单位进行数据包的解码；
   > - DelimiterBasedFrameDecoder：以特殊的符号作为分隔来进行数据包的解码；
   > - FixedLengthFrameDecoder：以固定长度进行数据包的解码；
   > - LenghtFieldBasedFrameDecode：适用于消息头包含消息长度的协议（最常用）；
   >
   > 基于Netty进行网络读写的程序，可以直接使用这些Decoder来完成数据包的解码。对于高并发、大流量的系统来说，每个数据包都不应该传输多余的数据（所以补齐的方式不可取），LenghtFieldBasedFrameDecode更适合这样的场景。

## 相关知识点

### **Zookeeper**

**主要应用场景**

1. **分布式锁** ： 通过创建唯一节点获得分布式锁，当获得锁的一方执行完相关代码或者是挂掉之后就释放锁。
2. **命名服务** ：可以通过 ZooKeeper 的顺序节点生成全局唯一 ID
3. **数据发布/订阅** ：通过 **Watcher 机制** 可以很方便地实现数据发布/订阅。当你将数据发布到 ZooKeeper 被监听的节点上，其他机器可通过监听 ZooKeeper 上节点的变化来实现配置的动态更新。

**[ ZooKeeper 重要概念解读](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_3-zookeeper-重要概念解读)**

*破音：拿出小本本，下面的内容非常重要哦！*

[3.1. Data model（数据模型）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_31-data-model（数据模型）)

ZooKeeper 数据模型采用层次化的多叉树形结构，每个节点上都可以存储数据，这些数据可以是数字、字符串或者是二级制序列。并且。每个节点还可以拥有 N 个子节点，最上层是根节点以“/”来代表。每个数据节点在 ZooKeeper 中被称为 **znode**，它是 ZooKeeper 中数据的最小单元。并且，每个 znode 都一个唯一的路径标识。

强调一句：**ZooKeeper 主要是用来协调服务的，而不是用来存储业务数据的，所以不要放比较大的数据在 znode 上，ZooKeeper 给出的上限是每个结点的数据大小最大是 1M。**

从下图可以更直观地看出：ZooKeeper 节点路径标识方式和 Unix 文件系统路径非常相似，都是由一系列使用斜杠"/"进行分割的路径表示，开发人员可以向这个节点中写人数据，也可以在节点下面创建子节点。这些操作我们后面都会介绍到。

![ZooKeeper 数据模型](https://snailclimb.gitee.io/javaguide/docs/system-design/distributed-system/zookeeper/images/znode-structure.png)

[3.2. znode（数据节点）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_32-znode（数据节点）)

介绍了 ZooKeeper 树形数据模型之后，我们知道每个数据节点在 ZooKeeper 中被称为 **znode**，它是 ZooKeeper 中数据的最小单元。你要存放的数据就放在上面，是你使用 ZooKeeper 过程中经常需要接触到的一个概念。

[3.2.1. znode 4种类型](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_321-znode-4种类型)

我们通常是将 znode 分为 4 大类：

- **持久（PERSISTENT）节点** ：一旦创建就一直存在即使 ZooKeeper 集群宕机，直到将其删除。
- **临时（EPHEMERAL）节点** ：临时节点的生命周期是与 **客户端会话（session）** 绑定的，**会话消失则节点消失** 。并且，**临时节点只能做叶子节点** ，不能创建子节点。
- **持久顺序（PERSISTENT_SEQUENTIAL）节点** ：除了具有持久（PERSISTENT）节点的特性之外， 子节点的名称还具有顺序性。比如 `/node1/app0000000001` 、`/node1/app0000000002` 。
- **临时顺序（EPHEMERAL_SEQUENTIAL）节点** ：除了具备临时（EPHEMERAL）节点的特性之外，子节点的名称还具有顺序性。

[3.2.2. znode 数据结构](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_322-znode-数据结构)

每个 znode 由 2 部分组成:

- **stat** ：状态信息
- **data** ： 节点存放的数据的具体内容

如下所示，我通过 get 命令来获取 根目录下的 dubbo 节点的内容。（get 命令在下面会介绍到）。

```shell
[zk: 127.0.0.1:2181(CONNECTED) 6] get /dubbo
# 该数据节点关联的数据内容为空
null
# 下面是该数据节点的一些状态信息，其实就是 Stat 对象的格式化输出
cZxid = 0x2
ctime = Tue Nov 27 11:05:34 CST 2018
mZxid = 0x2
mtime = Tue Nov 27 11:05:34 CST 2018
pZxid = 0x3
cversion = 1
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 1Copy to clipboardErrorCopied
```

Stat 类中包含了一个数据节点的所有状态信息的字段，包括事务 ID-cZxid、节点创建时间-ctime 和子节点个数-numChildren 等等。

下面我们来看一下每个 znode 状态信息究竟代表的是什么吧！（下面的内容来源于《从 Paxos 到 ZooKeeper 分布式一致性原理与实践》，因为 Guide 确实也不是特别清楚，要学会参考资料的嘛！ ） ：

| znode 状态信息 | 解释                                                         |
| -------------- | ------------------------------------------------------------ |
| cZxid          | create ZXID，即该数据节点被创建时的事务 id                   |
| ctime          | create time，即该节点的创建时间                              |
| mZxid          | modified ZXID，即该节点最终一次更新时的事务 id               |
| mtime          | modified time，即该节点最后一次的更新时间                    |
| pZxid          | 该节点的子节点列表最后一次修改时的事务 id，只有子节点列表变更才会更新 pZxid，子节点内容变更不会更新 |
| cversion       | 子节点版本号，当前节点的子节点每次变化时值增加 1             |
| dataVersion    | 数据节点内容版本号，节点创建时为 0，每更新一次节点内容(不管内容有无变化)该版本号的值增加 1 |
| aclVersion     | 节点的 ACL 版本号，表示该节点 ACL 信息变更次数               |
| ephemeralOwner | 创建该临时节点的会话的 sessionId；如果当前节点为持久节点，则 ephemeralOwner=0 |
| dataLength     | 数据节点内容长度                                             |
| numChildren    | 当前节点的子节点个数                                         |

[3.3. 版本（version）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_33-版本（version）)

在前面我们已经提到，对应于每个 znode，ZooKeeper 都会为其维护一个叫作 **Stat** 的数据结构，Stat 中记录了这个 znode 的三个相关的版本：

- **dataVersion** ：当前 znode 节点的版本号
- **cversion** ： 当前 znode 子节点的版本
- **aclVersion** ： 当前 znode 的 ACL 的版本。

[3.4. ACL（权限控制）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_34-acl（权限控制）)

ZooKeeper 采用 ACL（AccessControlLists）策略来进行权限控制，类似于 UNIX 文件系统的权限控制。

对于 znode 操作的权限，ZooKeeper 提供了以下 5 种：

- **CREATE** : 能创建子节点
- **READ** ：能获取节点数据和列出其子节点
- **WRITE** : 能设置/更新节点数据
- **DELETE** : 能删除子节点
- **ADMIN** : 能设置节点 ACL 的权限

其中尤其需要注意的是，**CREATE** 和 **DELETE** 这两种权限都是针对 **子节点** 的权限控制。

对于身份认证，提供了以下几种方式：

- **world** ： 默认方式，所有用户都可无条件访问。
- **auth** :不使用任何 id，代表任何已认证的用户。
- **digest** :用户名:密码认证方式： *username:password* 。
- **ip** : 对指定 ip 进行限制。

[3.5. Watcher（事件监听器）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_35-watcher（事件监听器）)

Watcher（事件监听器），是 ZooKeeper 中的一个很重要的特性。ZooKeeper 允许用户在指定节点上注册一些 Watcher，并且在一些特定事件触发的时候，ZooKeeper 服务端会将事件通知到感兴趣的客户端上去，该机制是 ZooKeeper 实现分布式协调服务的重要特性。

![watcher机制](https://snailclimb.gitee.io/javaguide/docs/system-design/distributed-system/zookeeper/images/watche%E6%9C%BA%E5%88%B6.png)

*破音：非常有用的一个特性，都能出小本本记好了，后面用到 ZooKeeper 基本离不开 Watcher（事件监听器）机制。*

[3.6. 会话（Session）](https://snailclimb.gitee.io/javaguide/#/docs/system-design/distributed-system/zookeeper/zookeeper-intro?id=_36-会话（session）)

Session 可以看作是 ZooKeeper 服务器与客户端的之间的一个 TCP 长连接，通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向 ZooKeeper 服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的 Watcher 事件通知。

Session 有一个属性叫做：`sessionTimeout` ，`sessionTimeout` 代表会话的超时时间。当由于服务器压力太大、网络故障或是客户端主动断开连接等各种原因导致客户端连接断开时，只要在`sessionTimeout`规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。

另外，在为客户端创建会话之前，服务端首先会为每个客户端都分配一个 `sessionID`。由于 `sessionID`是 ZooKeeper 会话的一个重要标识，许多与会话相关的运行机制都是基于这个 `sessionID` 的，因此，无论是哪台服务器为客户端分配的 `sessionID`，都务必保证全局唯一。

### 动态代理

**JDK动态代理机制**

+ 核心：InvocationHandler接口和Proxy类

  ```java
  	public static Object newProxyInstance(ClassLoader loader, Class<?> 			interfaces, InvocationHandler h) throws IllegalArgumentExcepiton {
      ......
  	}
  
  	public interface InvocationHandler {
          public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable;
      }
  ```

  

## 实现草稿

### 序列化模块

```java
public class KryoSerializer implements Serializer{
    private ThreadLocal<Kryo> kryoThreadLocal = ThreadLocal.withInitial(()->{
        Kryo kryo = new Kryo();
        kryo.register(RpcRequest.class);
        kryo.register(RpcResponse.class);
        kryo.setReferences(true);//是否关闭循环引用
        kryo.setRegistrationRequired(false);//是否关闭注册中心
        return kryo;
    });
    
    @Override
    public byte[] serialize(Object obj) {
        try (ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
             Output output = new Output(ByteArrayOutputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            //将Object写出
            kryo.writeObject(output, obj);
            kryoThreadLocal.remove();	
            return output.toBytes();
        }catch (Exception e) {
            throw new SerializeException("序列胡失败！");
        }
    }
    
    @Override
    public <T> T deserialize (byte[] bytes, Class<T> clazz){
 		try (ByteArrayInputStream byteArrayInputStream = new 			ByteArrayInputStream(bytes); Input input = new Input(byteArrayInputStream)) {
            Kryo kryo = kryoThreadLocal.get();
            // byte->Object:从byte数组中反序列化出对对象
            Object o = kryo.readObject(input, clazz);
            kryoThreadLocal.remove();
            return clazz.cast(o);
        } catch (Exception e) {
            throw new SerializeException("Deserialization failed");
        }
    }
}
```

