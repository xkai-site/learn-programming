# WebSocket快速入门

文章采用JavaScript进行演示，学习自b站技术蛋老师。

为了让服务器能够主动为客户端推送，使用WebSocket，而非HTTP。

```javascript
//服务器端
const WebSocket = require('ws');	//导入WebSocket
//wss是WebSocket实例，负责监听和管理所有客户端的连接（全局）
const wss = new WebSocket.Server({ port: 3001 });//创建一个WebSocket实例

wss.on('connection',ws=>{   //ws代表单个客户端的连接对象
    console.log('有个帅哥进来了');
    ws.on('message',message=>{
        console.log('收到消息：%s',message);
    });
    ws.on('close',()=>{
        console.log('有个帅哥走了');
    });
})
```

需要注意的是，我们会先创建一个ws/wss实例（多一个s更安全，类似https），这个实例管理所有客户的连接。一旦和用户产生连接，则会生成一个ws对象，我们只需要对这个对象进行操作，就可以通过对单一用户操作的管理，实现所有连接的相同管理。

```html
<!-- 客户端 -->
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    <script>
        const ws = new WebSocket('ws://localhost:3001');

        ws.addEventListener('open', () => {
            console.log('WebSocket连接已建立');
            ws.send('Hello, server!我是客户端');
        });
    </script>
</body>
</html>
```