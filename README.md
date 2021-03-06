# Mpush 2.0

## 功能
mpush是一套致力于用最简单,最快速的方式把消息从任何地方推送到指定的人的终端的系统,就像[server酱](http://sc.ftqq.com/3.version)一样  
只不过接收消息的不是微信而是任何可以建立websocket的客户端,就像下一点提到的安卓客户端,又或者是http服务器(webhook方式)  

## 安卓客户端仓库
[mpush-android-client(2.x)](https://github.com/kooritea/mpush-android-client/tree/2.x)

- 已经可以使用,功能正在完善中 
- 2.x版本mpush需要配合使用2.x版本的mpush-android-client

## 新特性
- 使用typescript重新编写
- 新增一对多按组推送
- 新增webhook的接入方式
- 新增websocket客户端双向传递消息

## 一、安装
- ### 稳定版  

在[发布页](https://github.com/kooritea/mpush/releases)下载并解压
仅需要ndoe环境(建议使用LTS 10.16.2或以上版本)

---

- ### 开发版

```bash
git clone https://github.com/kooritea/mpush.git
```
需要安装typescript，package的脚本是linux上的，windows需要自行修改

## 二、配置

直接编辑根目录的config.json
* 开发版只需编辑根目录的config.json
* 直接运行dist目录下编译后的应用则需要运行npm run build覆盖配置或手动编辑  

```javascript
{
    "HTTP_PORT":8090, // http服务器监听的地址
    "WEBSOCKET_PORT":8091, // websocket服务器监听的地址
    "TOKEN": "test", // 客户端接入需要与这个token匹配,webhook会带上这个token用于接收方校验
    "WEBHOOK_CLIENTS": [ // webhook配置,不使用可以留空(不要删除中括号[])
        {
            "NAME": "wh1", // 唯一名称,不应该与其他任何客户端同名,否则只有先接入的客户端能接收到消息
            "GROUP": "group2", // 组名,按组推送使用,
            "URL": "http://127.0.0.1:8092/mpush", // 请求地址
            "METHOD": "GET" // 请求方法
        },
        {
            "NAME": "TEST_WEBHOOK_POST",
            "URL": "http://127.0.0.1:8092/mpush",
            "METHOD": "POST"
        }
    ],
    "PUSH_TIMEOUT": 10000, // 等待回复确认的时间,超时未确认将会重发消息
    "PUSH_INTERVAL": 5000, // 重试间隔 推送失败或异常等待多少时间再重发
    "DEBUG": false, // 打印更多信息
}
```

## 三、运行

### 1、安装依赖

```bash
npm i
```
### 2、运行

- ### [发布页](https://github.com/kooritea/mpush/releases)下载的稳定版
```bash
node src/app.js
```

---

- ### 直接克隆或下载仓库的开发版
```bash
npm run dev
```

## 四、使用方式

### 1、客户端接入方式

#### (1) websocket

具体接入方式的实现由客户端实现，仅需要在客户端中填写name、group、token即可
#### (2) webhook

收到消息会以http请求的方式发送到指定的服务器，配置参考第二点中WEBHOOK字段

`额外信息(extra)是指指定的字段外的字段`
`额外信息可以配合客户端实现例如: 优先级,scheme等功能`

```javascript
// get
// 所有额外信息都会平铺到url上,但额外信息如果是多层对象则不会继续平铺,例如下面的a字段和b字段,b是一个编码的JSON字符串{hello:'world'}

'/mpush?token=test&sendType=personal&target=wh1&fromMethod=websock&fromName=anonymous&mid=111111111111&text=text10&desp=desp10&a=233&b=%7B%22hello%22:%22world%22%7D'


// post
// 这里的额外信息会全部放到message.extra字段,没有额外信息字段的时候extra为空对象
{
    token: config.TOKEN,
    sendType: "personal" | "group",
    target: name | group name,
    from: {
        method: 'websocket' | 'curl',
        name: name | 'anonymous'
    },
    mid: timestamp,
    message: {
        text: string,
        desp: string,
        extra: {
            a: 233,
            b: {
                hello: 'world'
            }
        }
    }
}
```

接收到webhook请求后 需要返回mid,不需要json格式,直接响应mid字符串

```javascript
GET 
-> mid

```

### 2、发送消息

#### (1) 使用http请求发送消息，可以使用GET和POST方法,接收text和desp两个字段，text一般用作title，参考server酱,除了text和desp参数,其他参数都会被放到extra字段中返回给客户端

格式

```bash
curl http://${HOST}:${HTTP_PORT}/${name}.${type:'send' | 'group'}?text=${text}&desp=${desp}&a=233&b=%7B%22hello%22:%22world%22%7D
```

其中`type`为`send`的时候name为客户端的name  
为`group`的时候`name`为`group name`,即一对多推送

例如  
向name为`kooritea`的客户端推送消息text为`hello`,desp为`world`的消息

```bash
curl http://HOST:HTTP_PORT/kooritea.send?text=hello&desp=world
```

向所有属于`kgroup`组的客户端发送消息

```bash
curl http://HOST:HTTP_PORT/kgroup.group?text=hello&desp=world
```
---

```javascript
// response
{
    cmd: 'MESSAGE_REPLY',
    data: {
        "client1 name": "ready",
        "client2 name": "ok",
        "client3 name": "no",
        "client4 name": "wait",
        "client5 name": "timeout",
    }
}
```
|status | 意义|
|------|---|
|ready | 消息队列中还有未发送的消息,正在等待前面的消息推送完成|
|ok | 推送成功|
|no | 推送失败,没有找到该客户端(只会出现在一对一推送)|
|wait | 正在等待回复确认|
|timeout | 首次推送超过时间未回复确认,该消息会等待重发|

注意: 假如该消息目标未进行第一次连接或未在config定义wenhook,则会默默失效

#### (2) 通过websocket客户端发送推送消息请求

这部分会由客户端实现(具体实现方式可以看下面的开发文档),用户只需要选择目标和内容

## 五、websocket客户端开发

[通信方式](./WSCLIENT_DEV.md)