# WebSocket

有些场景需要服务端主动源源不断地把数据发给客户端，比如游戏，即时通信等。

使用Http达到这个效果，本质上是客户端给服务端发：

1. 客户端定时发http请求到服务器（伪服务器推），定时轮询
   消耗带宽，明显卡顿
2. 长轮询（将客户端HTTP请求的超时设置得比较大）

那能不能让服务器主动给客户端发呢？其实可以，TCP协议是全双工的，而Http基于tcp只用于浏览文字等半双工操作，大概由于大部分场景已经使用HTTP，因此基于tcp写了个新协议WebSocket。

新协议WebSocket：
当客户端发送时确认使用Http还是WebSocket，需要发送WebSocket时需要携带请求头：

```
//请求
Connection:Upgrade  
Upgrade:websocket
Sec-WebSocket-Key:[Base64编码]
```

```
//响应
HTTP/1.1 101 Switching Protocols
Sec-WebSocket-Accept:[解码后的Base64]
Upgrade:websocket
Connection:Upgrade
```

传输的数据包在WebSocket中称为帧：

- opcode：定义帧的数据类型

- paylad：定义数据帧的长度
- payload：实际传输的数据

Java例子：

```java
public class MyWebSocketHandler extends TextWebSocketHandler {

    // 保存所有活跃连接
    private static final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String id = session.getId();
        sessions.put(id, session);
        session.sendMessage(new TextMessage("连接已建立（你的ID: " + id + "）"));
    }

    // 收到客户端消息时触发（真正的双向通信）
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        String clientMsg = message.getPayload();
        System.out.println("收到消息: " + clientMsg);

        // 主动向客户端推送（无需客户端请求）
        session.sendMessage(new TextMessage("服务器实时响应: " + clientMsg));

        // 示例：向所有连接广播
        broadcast("有人说了: " + clientMsg);
    }

    // 服务器主动推送方法（全双工核心）
    public void broadcast(String msg) {
        sessions.values().forEach(session -> {
            try {
                session.sendMessage(new TextMessage(msg));
            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }
}
```

