# 云纸条规格说明



项目组成共分为3个模块

- 前端UI
- 后端CGI
- 文档及素材



## 模块与实现

### 前端UI

项目技术栈使用vue-cli中的webpack模板，使用`vue-router`构建SPA。

功能模块划分如下：

- 导航：指导用户创建/加入演示间
- 创建演示间：创建一个演示间，可将笔记绘制在屏幕上
- 分享演示间：可将自己创建的演示间由某种方式分享给其他人
- 进入其他人创建的演示间：其他人可以加入有别人创建的演示间

功能实现如下：

- 首页承担导航作用，拥有2个超链接来指导用户创建/加入演示间
- 创建演示间会随机生成一个token
- 演示间创建完成后，在演示间中提示用户可以复制token给其他人
- 其他人拿到token，可通过首页的加入演示间，输入token自动跳转至演示间
- 仅创建者有演示间绘制能力

实现细节如下

- token随机生成，为16位随机字符（数字+字母，不区分大小写），每隔4位有一个`-`分割
- 演示间地址包含token，用户直接通过URL也依旧可进入演示间
- 演示间创建者：
  - 进入演示间后，向服务器发送token，建立websocket链接。
  - 在Canvas触发的“可供绘制的事件”会通过websocket事实send到服务器
- 演示间观看者
  - 进入演示间后，向服务器发送token，建立websocket链接。
  - 接受服务器发来的已有的绘制事件，并绘制在canvas中
  - 通过websocket实时接收由服务器发来的数据，绘制在canvas上
- 创建者与观看者的canvas应该是同步的

### 后台CGI

后台CGI仅提供数据，不包括渲染页面模板。

功能模块划分如下：

- 生成token
- 建立websocket链接
- 接受绘制事件，广播给同token的所有人
- 暂存所有绘制事件

功能实现如下：

- 接受创建token请求，创建一个token返回
- 第一个使用token的用户具有发送绘制事件的权限
- 接受拥有发送绘制事件的连接发来的绘制事件，存储，并广播给所有同token的所有连接
- 接受由客户端发送来的token，建立websocket协议
- 建立完成后，首先发送服务器已有该token的所有绘制事件

## 工作流

由主要2条工作流，可互不影响可同时进行：

- 前端UI
- 后台CGI

### 前端UI

1. 设计页面设计稿
2. 通过HTML+CSS切出
3. 用Vue进行组件化
4. 用Vue拼装组件构建页面

### 后台CGI

这个部分，一个人使劲写就行了。。。

### 接口文档

所有接口使用JSON格式进行传送，所有字母均使用小写。

接口包含以下2个通用字段：

- errcode：整数数字，0表示正常完成，其他值均表示出错
- errmsg：对该次请求的错误描述（用人类可读的语言进行描述）

后台接口如下：

#### 创建房间

- 接口URL：/token/create/:passwd？
  - passwd：可选，为房间设置的密码
- 调用方法：GET
- 接口返回：
  - token：请求创建后的房间token
  - errcode
  - errmsg

此接口会设置客户端cookie，请确保ajax请求中`withCredentials`为`true`。

请求此接口后，客户端可以直接向服务器通过`socket.io`发送websocket连接请求，服务器会监听`message`事件并广播。

#### 进入房间

- 接口URL：/websocket/connect/:token/:passwd？
  - token：必选，要连接的房间token
  - passwd：可选，进入房间的密码
- 调用方法：GET
- 接口返回：
  - errcode
  - errmsg
  
此接口会设置客户端cookie，请确保ajax请求中`withCredentials`为`true`。

请求此接口后，客户端可以直接向服务器通过`socket.io`发送websocket连接请求，并监听服务器的`message`事件

#### 获取房间列表

- 接口URL：/websocket/getrooms
- 调用方法：GET
- 接口返回：
  - rooms：数组，包含房间数据对象，对象属性如下：
    - id：房间ID
    - token：房间的token
    - passwd：Boolean类型，房间是否有密码
  - errcode
  - errmsg

### 其他接口

利用ws与向服务器发送的绘制事件数据结构如下：
```js
{
  timestamp:Int //事件发送的时间戳
  event_type:String[start|move|end] //发送的事件类型
  x:Float //坐标发生的X轴百分比位置，一个小于1的数字，小数点保留后4位
  y:Float //坐标发生的y轴百分比位置
  width:Int //对方画板的宽
  height:Int //对方画板的高
  pen: {
    type: String[pen|type] //笔类型
    size: Number //笔的大小 px
    [color=#000]: //笔的颜色
    [opacity=1.0]: //笔的透明度
  }
  [options]:{}    //该事件的其他描述
}
```
