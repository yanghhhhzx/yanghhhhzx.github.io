# 前言
GitHub项目源码展示：**[YChat](https://github.com/yanghhhhzx/YChat)**

我自己设计并实现了一个IM的即使通信系统，并内嵌了动态或朋友圈的功能，在这里展示我的一部分设计内容以及代码实现：

大多数的Javaweb开发新手，应该都做过商城、外卖等等项目，用到的很多技术和解决方法都类似，我也做过这些项目，所以在下面文章中，那些**千篇一律的部分我就不过多展示**了，我只写别人项目可能会用的少的技术，以及我自己独立思考的内容。

项目设计了3个模块，以微服务方式部署：用户微服务、聊天微服务、朋友圈微服务。
# 用户系统
## 用户信息
普通的用户系统做过很多次了，如果想弄点不一样的，可以考虑用户数据非常的情况。这时候为了避免单表索引过大，可以利用分库分表来提升性能，就可以引入ShardingSphere。

数据库存储选型：MYSQL。（这里我认为直接用TiDB更好，因为它自带分库分表。但因为市面上Mysql用得比较多，所以使用MySQL可能更加契合市场）。

对于用户信息，可以采用基础分片策略就行，因为用户登录是根据用户账号登录的，所以就以用户账号分片，我使用yaml文件来进行配置，下面是我的分片策略展示，只展示一次，后面再遇到分库分表就不再展示了：
``` yaml
dataSources:
  ds_0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://192.168.232.128:3306/ychat_user_0?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai
    username: root
    password: 123

  ds_1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://192.168.232.128:3306/ychat_user_1?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai
    username: root
    password: 123

rules:
  - !SHARDING
    tables:
      user:
        actualDataNodes: ds_${0..1}.user_${0..1}
        databaseStrategy:
          standard:
            shardingColumn: username
            shardingAlgorithmName: user_database_hash_mod
        tableStrategy:
          standard:
            shardingColumn: username
            shardingAlgorithmName: user_table_hash_mod

    shardingAlgorithms:
      user_database_hash_mod:
        type: HASH_MOD
        props:
          sharding-count: 2

      user_table_hash_mod:
        type: HASH_MOD
        props:
          sharding-count: 2
  - !ENCRYPT
# 加密,保证数据库泄露时的数据安全。
    tables:
      user:
        columns:
          phone:
            cipherColumn: phone
            encryptorName: phone_encryptor
# 加密算法选择，这里使用aes加密
    encryptors:
      phone_encryptor:
        type: AES
        props:
#   这个是秘钥，我记得aes的密钥是有安全性要求的，下面这个我乱写的，是不符合规范的，（但是位数是符合要求的：128位），仅做展示
          aes-key-value: yangyangzxyangzx
props:
  sql-show: true
```
## 用户关系
然后需要考虑的是用户关系表要如何分片，如果是一起的单表，那么只需要根据两个用户的id，分表创建索引就可以了。但是分库分表后，因为它**会根据查询字段再根据分片算法定位你的数据库，查询字段不在分库分表的分片算法中记录，那么就会导致全局扫描**，性能极差。

我认为最优的解决方法是对于**A与B的关系建立一条数据，然后B与A的关系再建立一条数据**，这样分片时只要根据对象1进行分片就行了。虽然需要多存一倍数据，但保证了速度，属于**空间换时间**。而且**拓展性很好**，如果后期需要给关系设置不同的状态或关系类型，比如设置成“关注”，也不需要改动数据库。


# 聊天系统
## 数据存储设计
### 用户聊天关系的存储
因为也考虑分库分表，其实完全可以用和用户关系表一样的方法，就是存两遍，但是也可以考虑一些其他的方式，拓展思维。我总共想了3中方法三点：排序就是我认为的它们的优先级顺序，第一种是最好的。
1. 存两遍。
2. 使用类似与MongoDB的json方式存储，不建立关系表。
3. 分库与表采用不同分片键（只能缩小范围，平均两种速率，作用不大）

我解释一下第二种方法和第三种方法的思路吧：
- 第二种： **直接在聊天表里添加member字段。里面记录群聊里面每个成员的账号信息**。这样要根据聊天查成员就非常快了，可以直接查聊天表就能获得所有成员的账号。同理，可以直接给用户加上一个chat的字段来记录他的群聊。也是一步到位，只需要查一个表。 缺点是操作该字段时会变得麻烦。只能使用select和updata，可以先将该字段读出来，转为集合类型再进行增删查改操作，再插入回去。（就类似于用MongoDB的方法存储）

- 第三种：分库分表采用不同的键值：因为在微信里面，是既需要根据用户查群聊，又根据群聊查用户，而不是两个参数同时出现。**复合分片是不行的**，因为每次的请求都是只有其中一个。我能想到的就是减少他需要查询的表，但是不能绝对确定。

因为第一种方式在用户关系表已经用过了，而且大部分情况都是采用关系表来处理关系，我想尝试一些别的方式，**在项目中我采用了方案二**。

### 存储每次登录需要更新的数据
先说明我的消息存储方式是吧消息存在一个消息表里面的，因为消息在被客户端读到本地后，服务器端的数据就可以不需要了，所以可能会经常删除，就不会数据量过大，我没有做分片（如果要分片，**要用群聊id分片**，不能用信息id分片）。

因为大部分的数据都是保存在客户端本地的，所以每次登录时，需要知道客户端的哪些数据有改动，再对客户端增量同步，而不是每次登录都进行全量同步（极度损耗性能）。

前端只需要登录后发起建立连接的请求就行，增量同步会在建立连接时就自动执行。

方案一：

1. 对于每个用户，可以在redis里面去维护一个LastTime，用来记录他的最后登录时间。
2. 对于每个用户在离线接收到消息时，会维护一个String类型数据实现的集合（读取时需要进行转换）用于记录他离线时更新信息的群聊的id。 这样可以利用复合索引，利用复合索引的前缀特性，建立一个复合索引，第一个是群聊id，第二个是发送时间，**这两个顺序不能改，不然会导致索引失效**。

方案二：

上面是一种解决思路，但是还有第二种方案，就是每个不维护时间+群聊，转而维护一个信箱，用来存放他的未读消息id，这样的优点是查询的时候可以直接定位到消息，不需要根据时间筛选，但是这样每个用户都需要一个信箱，这个信箱怎么设计，怎么实现是一个问题。

**我采用了方案一实现**。

## Java代码部分：WebSocket通信
实现即使聊天系统，只用http协议进行服务器和客户端的数据传输可行吗？因为http是一个请求-相应模型。如果要这样做，往往需要客户端主动得去刷新页面，才能得到新的用户信息，这样做会非常麻烦，且不能即使收到新的信息。要想即使收到新的信息，就需要使用WebSocket建立连接进行实时通信。

直接使用WebSocket因为没有高层级的协议，所以需要自定义应用之间所发送消息的语义，自己去对消息解析规范和解析。也可以使用可以利用Stomp作为集成，Spring社区提倡将Stomp作为WebSocket的子协议。（Apache的一位开发人员Justin Bertram认为）

在Java里建立Websocket得方法有很多，**一是利用Java原生得nio包，二是引入Netty这个网络高性能框架**。我是使用Netty来创建的Websocket的，因为Netty+WebSocket如何将Stomp作为子协议我没有找到适用的范例，不知道是否可行，且我看到Apache的开发人员Justin Bertram认为：“从功能契合的意义上看，Netty不支持STOMP协议”。所以我没有尝试使用Stomp，而是选择使用自己去规定消息格式，然后去按照规定去解析，约定大于配置。

### Websocket如何验证登录用户
我之前在B站上看到某位老程序员自己开发的IM通信系统，他用token来传递用户信息，但是因为ws连接跟平常http不太一样，他直接把token放到url里去了。**我认为这种做法是不合理的**。

虽然只要客户端拿到token，它就在客户端可查看得到。但是从网络安全角度看，URL在路由过程中可能会被某些网络设备查看，我认为还是放在消息体里面更加安全且优雅。其实我们要知道，在请求建立Websocket连接时，是采用http协议得，所以我们只需要在客户端请求建立连接时的请求头放入token就行了。以使用责任链模式，只要在解析消息之前加入解析http消息的handle就可以了。handle具体代码如下（有两个,第一个是解析token），里面包括了断开连接的操作：

#### FilterHandler:
``` java
package com.ychat.chat.websocket;

import com.ychat.chat.utils.JwtTool;
import com.ychat.common.exception.UnauthorizedException;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.HttpRequest;
import org.springframework.stereotype.Component;


@Component
public class FilterHandler extends ChannelInboundHandlerAdapter {

    private final JwtTool jwtTool;

    public FilterHandler(JwtTool jwtTool) {
        this.jwtTool = jwtTool;
    }
    // 已测试通过
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof HttpRequest) {
            HttpRequest request = (HttpRequest) msg;
            // 3.获取请求头中的token
            String token = null;
            String headers = request.headers().get("authorization");
            if (!(headers ==null)) {
                token = headers;
            }
            // 4.校验并解析token
            Long userId = null;
            try {
                userId = jwtTool.parseToken(token);
            } catch (UnauthorizedException e) {
                // 如果无效，拦截
                System.out.println("token无效");
//                channelGroup.remove(ctx);
                //改用concurrentHashmap了，上面那个不需要了,因为现在加入channels是在后面，在这里就不需要移除了
                super.channelInactive(ctx);
            }
            // 5.如果有效，传递用户信息
            String userinfo = userId.toString();
            request.headers().set("user-info",userinfo);
            super.channelRead(ctx, request);
        }
        else {
            super.channelRead(ctx, msg);
        }
    }
}

```
#### HttpRequestHandler:
``` java
package com.ychat.chat.websocket;

import com.ychat.chat.config.WebSocketConfig;
import com.ychat.chat.domain.Message;
import com.ychat.chat.service.MessageService;
import com.ychat.chat.utils.ChannelContext;
import com.ychat.chat.utils.ChatRedis;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.codec.http.HttpRequest;
import org.springframework.stereotype.Component;
import com.ychat.common.utils.transition.Transition;

import java.util.List;

import static com.ychat.chat.websocket.MyWebSocketHandler.channels;

/**
 * SimpleChannelInboundHandler用于处理入站请求
 */
@Component
public class HttpRequestHandler extends ChannelInboundHandlerAdapter {

    //如果在websocket中使用new来创建HttpRequestHandler的话，这里不能使用autowire注解！！
    //
    private final ChatRedis channelRedis;

    private final MessageService messageService;

    public HttpRequestHandler(ChatRedis channelRedis, MessageService messageService) {

        this.channelRedis = channelRedis;
        this.messageService = messageService;
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
        if (msg instanceof HttpRequest) {
            HttpRequest request = (HttpRequest) msg;
            // 1.获取请求头信息
            String userInfo = request.headers().get("user-info");
            System.out.println("获取到user"+userInfo);

            //把userid放入channel的上下文中，后面发送的时候要用
            //下面这个ChannelContext是我自己定义的！不是netty包下的那个。
            ChannelContext.addcontext(ctx.channel(),"userId",userInfo);
            System.out.println("保存成功");

            //加入ConcurrentHashMap,使用用户id作为key
            channels.put(ChannelContext.getcontext(ctx.channel(),"userId"),ctx);

            // 将channel和userid的对应关系保存到redis
            channelRedis.addPost(userInfo, WebSocketConfig.websocketPost);
            System.out.println("post已保存到redis");

            //**************************************************
            //在解析http请求时，顺便获取未读信息
            //放在这里。保证他是只有在建立连接时会执行（因为只有建立连接是http）
            //**************************************************

            List<String>chatIds =channelRedis.getUnReadChat(userInfo);//获取chat
            List<Long> chatList=Transition.StringListToLongList(chatIds);//转换chat
            String time=channelRedis.getLastTime(userInfo);//获取time
            List<Message>messages= messageService.getNewMessagesFromMysql(chatList,time);//获取message

            this.sendMessage(ctx,messages);
        }

        super.channelRead(ctx, msg);
//        ctx.fireChannelRead(msg);
    }
    // 客户端与服务器建立连接的时候触发
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("通道开启！");
        super.channelActive(ctx);
    }

    //将未读信息发送过去客户端
    private void sendMessage(ChannelHandlerContext ctx, List<Message> msg) {
        for (Message msg1 : msg) {
            ctx.channel().writeAndFlush(msg1);
        }
    }

    // 客户端与服务器关闭连接的时候触发
    @Override
    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        System.out.println("断开连接：通道关闭！");
        String userId=ChannelContext.getcontext(ctx.channel(),"userId");
        channelRedis.removeUnReadChat(userId);//删除UnReadChat
        channelRedis.setLastTime(userId);//改LastTime
        channels.remove(ctx.channel().toString());
        super.channelInactive(ctx);
    }

}
```
问为什么不在channelActive里面去解析？因为channelActive参数里面没有请求消息。

### 后端整个消息处理链的流程
```Java
package com.ychat.chat.config;

import com.ychat.chat.service.ChatService;
import com.ychat.chat.service.MessageService;
import com.ychat.chat.service.Producer;
import com.ychat.chat.utils.ChannelContext;
import com.ychat.chat.utils.ChatRedis;
import com.ychat.chat.utils.JwtTool;
import com.ychat.chat.websocket.FilterHandler;
import com.ychat.chat.websocket.HttpRequestHandler;
import com.ychat.chat.websocket.MyWebSocketHandler;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.stream.ChunkedWriteHandler;
import lombok.RequiredArgsConstructor;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Configuration;
import javax.annotation.PostConstruct;


@Configuration
@RequiredArgsConstructor
public class WebSocketConfig {

    private final ChatRedis chatRedis;
    private final ChannelContext channelContext;
    private final MessageService messageService;
    private final ChatService chatService;
    private final JwtTool jwtTool;
    private final Producer producer;
    @Value("${websocket.port}")
    public static String websocketPost;

    // 在应用启动时启动Netty服务器
    @PostConstruct
    public void startNettyServer() {
        //主线程池：负责接收客户端的连接请求
        EventLoopGroup bossGroup = new NioEventLoopGroup();
        //从线程池：负责处理已经建立连接的客户端的读写请求
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        //创建服务驱动，服务器
        ServerBootstrap serverBootstrap = new ServerBootstrap();
        serverBootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) throws Exception {
                        ChannelPipeline pipeline = ch.pipeline();
                        /**
                         *  当有消息进来时，他会根据pipeline一步一步往下走，执行完一个传递给下一个ChannelHandler
                         */
                        // Http编解码器
                        pipeline.addLast(new HttpServerCodec());

                        // 3. 添加ChunkedWriteHandler（可选，但如果你需要发送大消息，它很有用）
                        pipeline.addLast(new ChunkedWriteHandler());

                        // 2. 添加一个用于处理大请求的聚合器（可选，取决于你的需求）
                        pipeline.addLast(new HttpObjectAggregator(8192));// 例如，最大请求体大小为8KB

                        //*******************************************************
                        //*   下面两个是我自定义handler，如果token检验在网关则去掉第一    *
                        //*******************************************************
                        //完成过滤器功能，将token检验并转为userinfo，测试通过
                        pipeline.addLast("filter",new FilterHandler(jwtTool));
                        //用于获取http请求的请求头中的userinfo
                        pipeline.addLast("http", new HttpRequestHandler(chatRedis,messageService));


                        // 处理WebSocket升级握手、Ping、Pong等消息
                        // 将http协议升级为websocket
                        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
                        // 自定义WebSocket处理器

                        pipeline.addLast(new MyWebSocketHandler(messageService, chatService, chatRedis,producer));


                    }
                });
        //绑定端口启动netty服务
        serverBootstrap.bind(Integer.parseInt(websocketPost))
                .addListener((ChannelFutureListener) future -> {
                    if (future.isSuccess()) {
                        System.out.println("netty服务开启成功");
                    } else {
                        System.err.println("netty服务开启失败");
                    }
                });
    }
}
```
其中的MyWebSocketHandler就是真正处理消息的了。

### 跨越不同的websocket实例通信
在每个ws实例中，都有一个Channelgroup的static静态属性用来保存连接的Channel，在Channel里面添加一个名为userid的atrrikey字段，用来记录该Channel对应的客户。

同时在redis中，**根据userid存储**，每个用户所在实例的端口，以及ChannelId。因为连接信息是短暂存在的，所以可以用redis存储。
当需要向别人发送消息时，直接在redis里找，如果找到了就说明他在线，直接向对应端口他发送一个http请求，让他在websocket中发一条信息。
### 核心消息处理Hanndle
这是处理具体逻辑的客户收到的消息的，里面的设计可以看我另一篇关于concurrentHashMap的文章。
```java
package com.ychat.chat.websocket;

import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.ychat.chat.config.WebSocketConfig;
import com.ychat.chat.domain.Chat;
import com.ychat.chat.domain.Message;
import com.ychat.chat.domain.MessageToOne;
import com.ychat.chat.service.ChatService;
import com.ychat.chat.service.MessageService;
import com.ychat.chat.utils.ChatRedis;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.websocketx.TextWebSocketFrame;
import org.springframework.beans.BeanUtils;
import org.springframework.stereotype.Component;
import com.ychat.chat.utils.ChannelContext;
import com.ychat.common.utils.transition.Transition;
import com.ychat.chat.service.Producer;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;

@Component
/*
  只处理收发消息，其他的我都写在httpHandle
 */
public class MyWebSocketHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    //新的用来记录channel的，使用用户id做key
    //使用用户id来做key的话，用户就不能同时在多个地方同时登录了。
    //如果要多个地方同时登录就得四合院channel做key，因为我暂时没有考虑多端登录就先用用户id
    public static ConcurrentHashMap<String, ChannelHandlerContext> channels = new ConcurrentHashMap<>();
//已经无用
//    public static List<ChannelHandlerContext> channelGroup;

    private final MessageService messageService;

    private final ChatService chatService;

    private final ChatRedis chatRedis;

    private final Producer producer;

    public MyWebSocketHandler(MessageService messageService, ChatService chatService, ChatRedis channelRedis, Producer producer) {

        this.messageService = messageService;
        this.chatService = chatService;
        this.chatRedis = channelRedis;
        this.producer = producer;
    }

    /**
     * 已经通过拦截器拿到用户id了，放在ThreadLocal，现在需要把他放在channel里
     * （但是没有拿到账号，需要根据id查账号）
     */
    // 服务器接受客户端的数据信息
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        System.out.println("服务器收到的数据：" + msg.text());
        //获取根据msg获取message
        Message message=Message.getMessageByMsg(ChannelContext.getcontext(ctx.channel(),"userId"),msg);
        //保存到mysql
        messageService.saveMessageIntoMysql(message);
        System.out.println("获取到群聊id" + message.getChat());

        //1.根据chat在数据库中找到该群聊的所有成员。
        QueryWrapper queryWrapper = new QueryWrapper();
        queryWrapper.eq("chat", message.getChat());
        Chat chat = chatService.getOne(queryWrapper);

        //这里存储采用的是方法2，利用字段member存储成员，获取改群聊的成员id
        List<String> memberIds = Transition.StringToList(chat.getMember());
        System.out.println("查询群聊到成员" + memberIds);

        System.out.println("开始查询在线成员");
        // 2.根据将群聊成员列表遍历，根据redis查找在线的成员的channel。
        List<ChannelHandlerContext> channelHandlerContexts = new ArrayList<>();
        for (String memberId : memberIds) {
            String post = chatRedis.getPost(memberId);
            //利用RocketMQ发送异步消息
            MessageToOne messageToOne=new MessageToOne();
            //复制一个
            BeanUtils.copyProperties(message,messageToOne);
            messageToOne.setToOne(memberId);
            if (post != null) {producer.asyncSend(WebSocketConfig.websocketPost,messageToOne);}
        }
        //3.对信息包含的群聊的所有成员的未读群聊表进行更新（已经登录的人员会在断开连接时清除）
        for (String memberId : memberIds) {
            if (message.getChat() != null) {
                chatRedis.addChatIntoUnRead(message.getChat(), memberId);
            }
        }
        super.channelRead(ctx, msg);
    }

    public static int sendMessageByUserId(MessageToOne messageToOne){
        if (channels.containsKey(messageToOne.getToOne())){
            Message message=new Message();
            BeanUtils.copyProperties(messageToOne,message);
            channels.get(messageToOne.getToOne()).writeAndFlush(message);
            return 1;
        }
        return 0;//发送失败,可能是目标刚好下线了
    }

}

```

_**版权声明：本文为博主原创文章，转载请附上原文出处链接和本声明。**_

# 朋友圈系统
我的朋友圈系统