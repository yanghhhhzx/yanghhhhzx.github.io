# ConcurrentHashMap在项目中的实际应用
## HashMap存储与List存储的对比

HashMap是基于key-value的结构来存储数据的，它只能根据key查value，不能根据value查key，想要拿到hashmap中的某个元素，就必须知道他的key。**知道key后可以快速查找到想要的value。**

List只存value，没有key的对象，想要拿到其中的某个元素，**只能去遍历List集合，找到符合条件的元素，效率不高。**

## YChat项目中的实际应用。
为了方便理解，我先展示我最初版的代码，其实我做了3版，下面两种都是我自己优化的，在网上其他项目上基本找不到类似做法：
1. 只有一个websocket，所有客户端都连接在一个实例上（能在网上看到的项目，绝大多数都是这样单个实例的）。
2. 将channelgroup的存储方式改为HashMap。
3. 实现跨越不同实例通信。 （想看这一版直接看我的Ychat项目，里面有完整代码）

### 我一开始的写法：
在YChat中，需要把用户连接的Channel保存起来，一开始，我是使用一个List类型来保存的。如下：
` public static List<ChannelHandlerContext> channelGroup;`
但是在为了在channel中记录用户信息，所以我在用户建立连接后，会把他的id放入channel的AttributeKey中，利用如下的方法：
```java
    //在channel中添加一个名为keyName的key并赋值为userId
    //在外部我保存的用户id的keyName为"userId"
    public static void addcontext(Channel channel, String keyName,String userId){
        AttributeKey key = null;
        if (AttributeKey.exists(keyName)){//如果已经有了，那就是他
            key = AttributeKey.valueOf(keyName);
        }
        else {//如果没有，那就创建一个名为"userId"的key
            key = AttributeKey.newInstance(keyName);
        }
        channel.attr(key).set(userId);
    }

    public static String getcontext(Channel channel, String key) {
        return channel.attr(AttributeKey.valueOf(key)).get().toString();
    }

```

然后为了有用户发送信息时，找到需要接受信息的对象，就需要如下遍历：
```java
        // 2.根据将群聊成员列表遍历，查找在线的成员的channel。
        List<ChannelHandlerContext> channelHandlerContexts = new ArrayList<>();
        for (String memberId : memberIds) {
            for (ChannelHandlerContext cha : channelGroup) {
                if (ChannelContext.getcontext(ctx.channel(),"userId").equals(memberId)) {
                    channelHandlerContexts.add(cha);
                    //这里是否break取决于你的系统是否考虑支持多端同时登录（同一个用户id，多个channel）。
                    //如果存在多端登录就不break，如果不支持就break。
                    break;
                }
            }
        }
```

### 问题分析
我在接受到用户发送信息时，需要先查到发往的群聊，再根据群聊拿到需要发送给的userId，然后需要通过遍历的方式找到channel，当channel过多是，就会导致查找速度很慢。这就可以用到map了。

### 改造
先引入hashmap
```java
 //新的用来记录channel的，使用用户id做key
 //使用用户id来做key的话，用户就不能同时在多个地方同时登录了。
 //如果要多个地方同时登录就得用channel做key，但多端登录还须考虑安全问题，因为我暂时没有考虑多端登录就先用用户id。
public static ConcurrentHashMap<String, ChannelHandlerContext> channels = new ConcurrentHashMap<>();
```
刚才的循环进行改造，我是先将需要发送的统计起来，再调用一个函数统一去进行发送的，这里其实直接writeAndFlush发送也行。
```java
        // 2.根据将群聊成员列表遍历，查找在线的成员的channel。
        List<ChannelHandlerContext> channelHandlerContexts = new ArrayList<>();
        for (String memberId : memberIds) {
            if (channels.get(memberId)!=null){channelHandlerContexts.add(channels.get(memberId));}
        }
```
然后把其他之前使用channelGroup的地方记得全部改为channels。

# 总结
## 优点
所谓Map，直接翻译过来是“地图”，而地图就是可以指明你的资源的存储地点，按图索骥，所以使用Map可以很方便的根据key快速定位到value，而HashMap是其中的一种。而ConcurrentHashMap可以加锁，在多线程条件下保证线程安全。

## 缺点
因为它刚需一个key，而且只能有一个key，不支持根据value查key。假设在上面的例子中，我既需要根据channelId查userId，又需要根据userId查channelId，有两种查询维度，就没办法了。这种情况下我能想到的办法就是建立两个hashmap，一个用channelId做key，一个用userId做key。但若是有N个查询条件呢？那我想只能用List集合来存了，然后在查找的时候做遍历。