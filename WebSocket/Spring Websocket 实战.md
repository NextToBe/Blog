# Spring WebSocket 实战 — 实现点对点通信



***本文将简单介绍通过使用 Spring 框架的 websocket 来实现点对点的安全通信***

 

##maven 依赖

本次的 *maven* 依赖主要是:

* *Spring Security*
* *Spring Websocket*
* *Spring Messaging*
* *Spring Security Messaging*



## Spring Websocket 配置

首先要说明的是本次实验使用的并不是纯正的 *websocket* 协议，而是基于 *websocket* 协议的 *STOMP* 协议，所以在使用上会有一定的偏差。之所以要使用 *STOMP* 而不使用 *websocket* 的原因跟我们使用 ***TCP*** 编程而不是直接使用 **套接字** 的原理是一样的。 ***STOMP*** 协议是在 ***websocket*** 上层的协议，它使用起来更加的方便。



首先我们需要对 *STOMP* 进行配置，如下：

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketSecurityConfiguration extends AbstractSecurityWebSocketMessageBrokerConfigurer {
  
    @Override
  public void registerStompEndpoints(StompEndpointRegistry registry) {
    registry.addEndpoint("/ws").setAllowedOrigins("*").withSockJS();
    registry.addEndpoint("/ws").setAllowedOrigins("*");
  }

  @Override
  public void configureMessageBroker(MessageBrokerRegistry registry) {
    registry.enableSimpleBroker("/channel", "/user");
    registry.setUserDestinationPrefix("/user");
  }

  @Override
  protected void configureInbound(MessageSecurityMetadataSourceRegistry messages) {
    messages.nullDestMatcher().authenticated().simpSubscribeDestMatchers("/channel/**").permitAll()
        .simpSubscribeDestMatchers("/user/**").authenticated();
  }

  @Override
  protected boolean sameOriginDisabled() {
    // disable CSRF
    return true;
  }

}

```

上面的注解 *EnableWebSocketMessageBroker*  不仅配置了 *WebSocket*，还配置了基于代理的 *STOMP* 消息， 它复写了 *registerStompEndpoints()* 方法：添加一个服务端点，来接收客户端的连接。将 *“/ws”* 路径注册为 STOMP 端点。这个路径与之前发送和接收消息的目的路径有所不同， 这是一个端点，客户端在订阅或发布消息到目的地址前，要连接该端点，即用户发送请求 ：*url  = /127.0.0.1:8080/ws* 与 *STOMP Server* 进行连接，之后再转发到订阅 *url*  。同时，它还复写了 *configureMessageBroker*() 方法：配置了一个 简单的消息代理，通俗一点讲就是设置消息连接请求的各种规范信息。



之后的方法 *configureInbound()* 定义了订阅的路径是否需要校验。 其中，空的订阅地址需要校验，而以 ***channel*** 开头的不需要校验，而以 ***user*** 开头的则又需要校验。这里的校验一般来说是指 ***Spring Security*** 。



最后一个方法是取消 ***CSRF*** ，否则客户端还需要额外传入 ***CSRF Token***



## 监听连接与订阅事件

在配置完成之后，我们将重点转移到我们需要实现的功能上来，我们需要实现 **点对点通信** ，所以必须知道 ***“谁是谁”***。 所以，我们必须在用户与服务器建立连接时去将它们甄别开来。

```java
@Service
@Slf4j
public class MessageWebSocketService {

  private static final String MESSAGE_DESTINATION = "/message";

  //这里的前缀需要 debug 才能确定
  private static final String SESSION_ID = "simpSessionId";

  private static final String USER = "simpUser";

  private static final String DESTINATION = "simpDestination";

  // 使用 ConcurrentHashMap 来记录用户与 Session 的映射关系
  private ConcurrentHashMap<String, String> userSessionMap = new ConcurrentHashMap<>();

  @Autowired
  IUserService userService;

  @Autowired
  IMessageService messageService;

  @Autowired
  SimpMessagingTemplate template;


  // 监听连接建立事件
  @EventListener
  public void handleSessionConnneted(SessionConnectedEvent event) {
    // 建立连接时记录映射关系
    userSessionMap.put(getUserLdapAccount(event), getSessionId(event));
  }

  // 监听订阅事件
  @EventListener
  public void handleSessionSubscribe(SessionSubscribeEvent event) {
    String destination = getDestination(event);
    String[] segs = destination.split("/");
    // 判断是否需要继续下一步
    if (!segs[0].equals("user")) {
      return;
    }

    String user = segs[2];
    String ldapAccount = getUserLdapAccount(event);
    if (!ldapAccount.equals(user)) {
      // 进一步保障安全, 防止恶意 JS 代码
      userSessionMap.remove(ldapAccount);
      throw new MyException(ErrorCode.NOT_AUTHORIZED, "您没有权限监听他人的端口");
    }

     UserVo userVo = userService.fetchUserByLdapAccount(getUserLdapAccount(event));
    if (userVo != null) {
      // 刚上线的用户需要收到所有未发送的通知
      List<MessageVO> messageVOS = messageService.fetchUnsendMessages(employeeVO.id);
      sendMessageToUser(employeeVO.ldapAccount, messageVOS);
    }
  }


  // 监听连接建立的事件
  @EventListener
  public void handleSessionDisconnet(SessionDisconnectEvent event) {
    // 从映射关系中删除
    if (!userSessionMap.containsKey(getUserLdapAccount(event))) {
      return;
    }

    userSessionMap.remove(getUserLdapAccount(event));
  }

  private String getSessionId(AbstractSubProtocolEvent event) {
    MessageHeaders headers = event.getMessage().getHeaders();
    return (String) headers.get(SESSION_ID);
  }

  private String getUserLdapAccount(AbstractSubProtocolEvent event) {
    // 本方法是从 header 中获取用户账号的实现, 这里使用的是 Ldap 账户体系实现，具体实现是
    // Spring Security 中的内容。
    MessageHeaders headers = event.getMessage().getHeaders();
    UsernamePasswordAuthenticationToken token = (UsernamePasswordAuthenticationToken) headers.get(USER);
    // 所有的用户体系都绕不开 Principal
    ldapUserDetailsImpl user = (LdapUserDetailsImpl) token.getPrincipal();
    return user.getUsername();
  }

  private String getDestination(AbstractSubProtocolEvent event) {
    MessageHeaders headers = event.getMessage().getHeaders();
    return (String) headers.get(DESTINATION);
  }


  // 发送消息的具体实现
  public boolean sendMessageToUser(String userLdapAccount, List<MessageVO> messageVOS) {
    // 只发送给当前在线的用户
    if (!userSessionMap.containsKey(userLdapAccount)) {
      return false;
    }
    try {
      // 框架会使用 user 和 destination 拼接订阅路径，事实上前端也需要按照这个规则进行订阅。
      // 之后会获取所有当前所有的订阅路径(连接状态下)并一一进行发送。
      template.convertAndSendToUser(userLdapAccount, MESSAGE_DESTINATION, messageVOS);
      
      // 标记这些消息为已经发送，不再发送
      return messageService.markAsSent(messageVOS.stream().map(MessageVO::getId).collect(Collectors.toList()));
    } catch (MessagingException ex) {
      log.error("sending to {} error.", userLdapAccount, ex);
      return false;
    }

  }
```



我们主要使用了 *Spring* 的 ***EventListener*** 机制来进行用户与 *Session* 的自我管理，如果你使用 ***websocket*** 协议，你可以来实现 ***WebsocketHandler*** 接口来达到同样的目的。



## 前端部分代码

之后就是前端的具体代码，用到了最主要的库是 ***StompJS***，使用它可以跟服务器进行完美的沟通。 这里只展示重要逻辑部分：

```javascript
function connect(event) {
    username = document.querySelector('#name').value.trim();

    if (username) {
        usernamePage.classList.add('hidden');
        chatPage.classList.remove('hidden');

        // 建立连接，触发 SessionConnectedEvent
        var socket = new SockJS('/ws');
        stompClient = Stomp.over(socket);

        stompClient.connect({}, onConnected, onError);
    }
    event.preventDefault();
}


function onConnected() {
    // 触发 SessionSubscribeEvent
    // Subscribe to the Public Channel
    stompClient.subscribe('/channel/chatRoom', onMessageReceived);

    stompClient.subscribe('/user/{yourUserName}/message', onMessageReceived);

    connectingElement.classList.add('hidden');
}


// 用来向服务器发送数据
function sendMessage(event) {
    var messageContent = messageInput.value.trim();

    if (messageContent && stompClient) {
        var chatMessage = {
            sender: username,
            content: messageInput.value
        };

        stompClient.send("/app/chat/sendMessage", {}, JSON.stringify(chatMessage));
        messageInput.value = '';
    }
    event.preventDefault();
}
```





## 结语

这里只简单的实现了 ***点对点通信***，但实际上的情况要远比这个复杂的多。下次有时间在写吧 :=|