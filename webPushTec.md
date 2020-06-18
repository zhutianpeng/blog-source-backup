---
title: 服务器推送技术
date: 2020-04-25 15:11:08
tags: web 
---

>为解决Web应用中server向client“主动”发起请求的需求，出现了服务器推送技术。本文对服务器推送技术进行分类，然后分析其具体应用场景，例如淘宝，微信的web端的扫码登录的流程。

# 1. 服务器推送技术分类

http协议为无状态，单向性的协议，必须有客户端发起请求，服务端接受请求，做出响应。在此特性下，要实现服务端“主动”发起的推送技术，有如下几种实现方式：

- Ajax短轮询
- Comet
    - Ajax长轮询
    - http streaming
- SSE
- WebSocket
- 基于MQ


## 1. Ajax短轮询
- 定义： 短轮询是指客户端定时向服务器发送ajax请求，服务器接到请求后马上返回响应信息并关闭连接。Ajax短轮询其实就是：“脚本发送的http请求。”
- 举例： 
    ```html
    <script type="javascript">
        showTime();
        function showTime() {
            $.get("showTime",function(data){
                console.log(data);
                $("#serverTime").html(data);
            })
        }
        setInterval(showTime,1000); //重点在于定时发送
    </script>
    ```

## 2. Comet
- 定义：Alex Russell（Dojo Toolkit 的项目 Lead）称基于 HTTP 长连接、无须在浏览器端安装插件的“服务器推”技术为“Comet”。
- 分类：
    - 基于 AJAX 的长轮询（long-polling）方式
    - 基于 Iframe 及 htmlfile 的流（streaming）方式

### 2.1 基于 AJAX 的长轮询（long-polling）方式
![轮询与长轮询](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/%E8%BD%AE%E8%AF%A2%E4%B8%8E%E9%95%BF%E8%BD%AE%E8%AF%A2.png)

- 定义：在普通轮询过程中，client接受响应的速度受制于定时器的发送频率。长轮询中，当server接受到client发送的请求后，hold住一段时间，若有消息，则立刻返回。client接受到响应（1）、或者超时（2）、或者异常（3）时，重新发起一次新的请求。
- 应用：在一些使用Pull模式消费的消息系统，会使用Long Polling技术进行优化。例如下图是MetaQ推送的服务端设计，实际是利用的长轮询的原理。

    ![metaQ的push](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/metaq%E7%9A%84push.png)

### 2.2 基于Iframe及htmlfile的流（streaming）方式
- 定义：iframe 是很早就存在的一种 HTML 标记， 通过在 HTML 页面里嵌入一个隐蔵帧，然后将这个隐蔵帧的 SRC 属性设为对一个长连接的请求，服务器端就能源源不断地往客户端输入数据。
- 实现方式：iframe 服务器端并不返回直接显示在页面的数据，而是返回对客户端 Javascript 函数的调用，服务器端将返回的数据作为客户端 JavaScript 函数的参数传递；客户端浏览器的 Javascript 引擎在收到服务器返回的 JavaScript 调用时就会去执行代码。（即是在返回的数据中嵌入JS脚本），例如：

```
&lt;script type="text/javascript"&gt;js_func(“data from server ”)&lt;/script&gt;
```

- 应用：Google 使用一个称为“htmlfile”的 ActiveX 解决了在 IE 中的加载显示问题，并将这种方法用到了 gmail+gtalk 产品中。

## 3. SSE （Server-Sent Events）
![SSE](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/SSE.png)
- 定义：服务器向客户端声明，接下来要发送的是流信息（streaming）。这时，客户端不会关闭连接，会一直等着服务器发过来的新的数据流。本质上是以流信息的方式，完成一次用时很长的下载。
- 应用：视频播放
- 实现方式，参考 阮一峰的[Server-Sent Events 教程](https://www.ruanyifeng.com/blog/2017/05/server-sent_events.html)


## 4. WebSocket
![http与WebSocket](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/websocket.png)

- 定义：WebSocket是一种网络传输协议，可在单个TCP连接上进行全双工通信，位于OSI模型的应用层。
- 特点：可以发送文本，也可以发送二进制数据。协议标识符是ws（如果加密，则为wss），服务器网址就是 URL。
- 基于client：javascript 和 server：java的实现：

1. maven添加websocket库

```
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-websocket</artifactId>
  <exclusions>
    <exclusion>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-tomcat</artifactId>
    </exclusion>
  </exclusions>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

2. server：

    有两种创建服务器端代码的方法：

- 注解方式(Annotation-driven): 通过在POJO加上注解, 开发者就可以处理WebSocket 生命周期事件。
- 实现接口方式(Interface-driven): 开发者可以实现Endpoint接口和声明周期的各个方法。

添加配置类 WebSocketConfig 代码如下:

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig {

    @Bean
    public ServerEndpointExporter serverEndpoint() {
        return new ServerEndpointExporter();
    }
}
```

添加处理连接和处理消息类 MyWebSocket

```java
@ServerEndpoint(value = "/websocket")
@Component
public class MyWebSocket {
    //在线人数
    private static int online = 0;
    //所有的对象，用于群发
    public static List<MyWebSocket> webSockets = new CopyOnWriteArrayList<>();
    //会话
    private Session session;

    public Session getSession() {
        return session;
    }

    //建立连接
    @OnOpen
    public void onOpen(Session session) {
        online++;
        webSockets.add(this);
        this.session = session;
    }

    //连接关闭
    @OnClose
    public void onClose() {
        online--;
        webSockets.remove(this);
    }

    //收到客户端的消息
    @OnMessage
    public String onMessage(String text, Session session) {
        return "client message: " + text;
    }
}
```

3. client:

```js
if (window.WebSocket) {
  // Create WebSocket connection.
  const socket = new WebSocket('ws://localhost:8080/websocket');
  // Connection opened
  socket.addEventListener('open', function (event) {
    socket.send('Hello Server!');
  });
  // Connection closed
  socket.addEventListener('close', function (event) {
    socket.send('I am leave!');
  });
  // Listen for messages
  socket.addEventListener('message', function (event) {
    console.log(event);
  });
  // 延时给服务端发送一条消息
  setTimeout(function() {
    socket.send('hello world!');
  }, 500);
}
```

## 5. 基于MQ
- 一般由webserver向webclient进行消息推送，中间并不包含第三者（中间件）。但是如对需要推送的消息具有更高层面的要求，例如 架构上松耦合、异步处理、消息管理、持久化、削峰填谷等。则可以利用消息中间件来完成消息的传递。而将webserver作为消息的生产者，发送消息经过MQ，传递到webclient的过程，就完成了基于MQ的服务器端的推送。
- 关于消息中间件可以参看这篇：[以activemq为例的消息中间件浅析](https://ztxpp.cc/2019/05/06/MQ/)
- 一种实现方式： js client(STOMP.js) + activeMQ + java server


# 2. 推送技术的对比分析
- Ajax短轮询：
    - 优点：不用配置，非常简单，就是一个定时器。
    - 缺点：
        1. 轮询的时间间隔需要进行仔细考虑。
        轮询的间隔过长，延迟高。会导致用户不能及时接收到更新的数据；轮询的间隔过短，server压力大。查询请求过多，增加服务器端的负担（每次轮询都会进行一次完整的HTTP请求，如果没有数据更新，相当于是一次“浪费”的请求）。

        2. 一旦遇到页面有大量任务或者返回时间特别耗时，页面可能会出现‘假死’，无法响应用户行为。

- Comet：
    - 优点：减少了不必要的http请求次数。
    - 缺点：连接挂起（hold）也会导致资源的浪费。
    - 分析：轮询与长轮询都是基于HTTP的，两者本身存在着缺陷:轮询需要更快的处理速度；长轮询则更要求处理并发的能力;两者都是“被动型服务器”的体现:服务器不会主动推送信息，而是在客户端发送ajax请求后进行返回的响应。而理想的模型是”在服务器端数据有了变化后，可以主动推送给客户端”,这种”主动型”服务器是解决这类问题的很好的方案。Web Sockets就是这样的方案

- SSE:
    - SSE单工，WebSocket双工
    - SSE 默认支持断线重连，WebSocket 需要自己实现。
    - 一般只用来传送文本，二进制数据需要编码后传送，WebSocket 默认支持传送二进制数据。
    - SSE使用 HTTP 协议，现有的服务器软件都支持。WebSocket 是一个独立协议。

- WebSocket:
    - 从互联网协议的角度看：之前三者都是应用层基于TCP的HTTP协议，WebSocket则是应用层的另一个基于TCP协议；
    - 是真正意义上的全双工通信。
    - 实现起来，与前三种相比较为复杂。

- MQ:
    - 从协议角度看，消息中间件实现的协议，例如：
        - AMQP（AdvancedMessage Queuing Protocol）即高级消息队列协议是应用层协议的一个开放标准
        - STOMP（Simple (or Streaming) Text Orientated Messaging Protocol），即简单的面向文本/流的消息协议，一般是Stomp Over Websocket，可以理解为基于WebSockets的一个协议。
    - 对于消息而言，可以满足更高层面的需求。

- 几种连接方式的比较：

    ![连接方式的比较](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/%E8%BF%9E%E6%8E%A5%E6%96%B9%E5%BC%8F%E6%AF%94%E8%BE%83.png)


# 3. 服务器推送举例分析
## 3.1 淘宝网页版扫码登录
- 需求分析：淘宝，京东平台网页版，具有扫码登录环节，需要手机扫码配合完成登录流程。手机登录成功后，server端是如何将消息推送到web client的呢？下面就此进行分析。
- 抓包分析：

    ![抓包分析](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/%E6%B7%98%E5%AE%9D%E6%89%AB%E7%A0%81%E7%99%BB%E5%BD%95.png)

    只针对server是如何推送消息到web client这一阶段，进行抓包分析，可知使用的是时间间隔为大约5s的Ajax短轮询。

- 全流程时序分析

    ![淘宝网页版扫码登录时序图](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/%E6%B7%98%E5%AE%9D%E6%89%AB%E7%A0%81%E7%99%BB%E5%BD%95%E6%97%B6%E5%BA%8F%E5%9B%BE.png)

    全流程时序分析不是本文的重点，大致分为三个阶段： 网页获取二维码阶段，网页短轮询阶段，手机扫码登录阶段。

    - 网页获取二维码阶段：web browser与 web server进行一次通信，网页显示出二维码信息。web server将此uuid信息返回给browser，并在缓存中以此 uuid为key进行保存，并且设置过期时间。
    - 网页轮询阶段： web browser 通过5s一次的短轮询，希望从web server中获取手机是否已经扫码完成登录的信息，当未过期，或者为获取到userId信息时，重复请求。
    - 手机扫码登录阶段： 手机端通过扫描二维码，获取到uuid和二维码验证信息，并辅以手机端登录态的token信息，向mobile server发起web端登录请求； server比对二维码信息后，恢复是否确认登录；手机端点击确认登陆后，server将此uuid做key, 用手机端登录态token中获取到的userId作value存入缓存中。并回复手机端登陆成功。
    - 网页端轮询阶段：网页端轮询请求至此，web server便可以由uuid获取到 userId的信息，登陆成功，返回web token，跳转至首页。

## 3.2 京东网页版扫码登录
代码分析，可知是使用的短轮询。

```js
<script language="JavaScript">
    function loginGetEid(count) {
        if(count >= 4) {
            return;
        }
        try {
            if(typeof(getJdEid) == "function") {
                getJdEid(function(eid,fp,udfp){
                    $("#eid").prop("value", eid);
                    $("#sessionId").prop("value", fp);
                });
            } else {
                count ++;
                setTimeout('loginGetEid('+count+')', 300);
            }
        }catch(e){
            $("#eid").prop("value", "unknown");
            $("#sessionId").prop("value", "unknown");
        }
    }

    setTimeout('loginGetEid(0)', 1000); //1s后执行此方法，可能为了节省轮询的资源（用户不会那么快扫码完成）
</script>
```

分析：淘宝网页版的扫码登录，选用了Ajax短轮询技术，完成server向web client的推送。淘宝大约是5s的短轮询，京东登录是大约3s的短轮询。选择使用Ajax短轮询的原因：

- 淘宝、京东网页版需要很高的浏览器兼容性。
- 与巨量用户在网页端的图片浏览和查询相比，3s和5s一次的短轮询占用带宽很小，而且扫码登录的实时性也不要求太高。在需求的范围内，节省资源。

## 3.3 微信网页版
微信网页版的扫码登录功能，由图可知，使用的是25s的长轮询。
![微信网页版扫码登录](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/%E6%9C%8D%E5%8A%A1%E5%99%A8%E6%8E%A8%E9%80%81/%E5%BE%AE%E4%BF%A1%E7%BD%91%E9%A1%B5%E7%89%88%E6%89%AB%E7%A0%81%E7%99%BB%E5%BD%95.png)

微信网页版通信功能，也是基于长轮询实现。并未采用websocket，个人认为是考虑到 兼容性，和一定程度的实时性。（并未求证，截止2020.4.27日，已经无法登录微信网页版）

## 3.4 webSocket应用
- 社交订阅
- 多玩家游戏
- 协同编辑/编程
- 点击流数据
- 股票基金报价w
- 体育实况更新
- 多媒体聊天
- 基于位置的应用
- 在线教育

上述应用领域，都是在考虑实时性较高的情况下，使用webSocket。

> 后文：同时代的技术，没有优劣之分，只有适不适合需求的区别。并非实时性越强越好，有时候需要考虑较多其他的因素。

参考：
- [Comet：基于 HTTP 长连接的“服务器推”技术](https://www.ibm.com/developerworks/cn/web/wa-lo-comet/#artrelatedtopics)
- [Server-Sent Events 教程](https://www.ruanyifeng.com/blog/2017/05/server-sent_events.html)
- [WebSocket 教程](https://www.ruanyifeng.com/blog/2017/05/websocket.html)
- [Java WebSocket教程](https://colobu.com/2015/02/27/WebSockets-tutorial-on-Wildfly-8/)
- [说说微信和淘宝扫码登录背后的实现原理](https://cloud.tencent.com/developer/article/1589934)