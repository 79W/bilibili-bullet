需求：实时获取直播间的弹幕

我们打开一个B站的直播间

```javascript
https://live.bilibili.com/23423267?session_id=6db8b01d511cae96d90825a02161af46_E97F63BD-7769-40B1-8239-2CFC7294000A&launch_id=1000218&hotRank=0&spm_id_from=333.1007.partition_recommend.content.click
```

然后我们删除 `?`即以后的所有内容 就剩下这个链接了

```javascript
https://live.bilibili.com/23423267
```

这里可以看到一串数字 这并不是真实的 直播间ID

#### 获取真实直播间ID

```javascript
// 注意ID
https://api.live.bilibili.com/room/v1/Room/room_init?id=23423267
```

这里需要填写个ID 这就是 网页URL 上面的数字

```javascript
async function getRoomId(id) {
  const res = await Axios.get(`https://api.live.bilibili.com/room/v1/Room/room_init?id=${id}`);
  return res.data;
}
```

返回值

```javascript
{
  "code": 0,
  "msg": "ok",
  "message": "ok",
  "data": {
    "room_id": 23423267,
    "short_id": 0,
    "uid": 431856380,
    "need_p2p": 0,
    "is_hidden": false,
    "is_locked": false,
    "is_portrait": false,
    "live_status": 1,
    "hidden_till": 0,
    "lock_till": 0,
    "encrypted": false,
    "pwd_verified": false,
    "live_time": 1650773392,
    "room_shield": 0,
    "is_sp": 0,
    "special_type": 0
  }
}
```

这里数据会反回真实的直播间ID 也就是：`room_id`

#### 获取WebSocket地址

我们有了真实的房间ID就需要获取 Socket 交互的地址

```javascript
// roomid ===》 room_id
https://api.live.bilibili.com/room/v1/Danmu/getConf?room_id=${roomid}&platform=pc&player=web
```

写个方法然后方便进行调用

```javascript
async function getWebSocketHost(roomid) {
  const res = await Axios.get(
    `https://api.live.bilibili.com/room/v1/Danmu/getConf?room_id=${roomid}&platform=pc&player=web`
  );
  return res.data;
}
```

返回值

```javascript
{
  "code": 0,
  "msg": "ok",
  "message": "ok",
  "data": {
    "refresh_row_factor": 0.125,
    "refresh_rate": 100,
    "max_delay": 5000,
    "port": 2243,
    "host": "broadcastlv.chat.bilibili.com",
    "host_server_list": [
      {
        "host": "hw-sh-live-comet-01.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      },
      {
        "host": "tx-bj-live-comet-04.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      },
      {
        "host": "broadcastlv.chat.bilibili.com",
        "port": 2243,
        "wss_port": 443,
        "ws_port": 2244
      }
      ],
      "server_list": [
      {
        "host": "119.3.126.45",
        "port": 2243
      },
      {
        "host": "49.232.64.111",
        "port": 2243
      },
      {
        "host": "broadcastlv.chat.bilibili.com",
        "port": 2243
      },
      {
        "host": "119.3.126.45",
        "port": 80
      },
      {
        "host": "49.232.64.111",
        "port": 80
      },
      {
        "host": "broadcastlv.chat.bilibili.com",
        "port": 80
      }
  	],
    "token": "_xRKFgXU73NHsSrvh4jAKiQ8CRFZMqao1r6xarlMgqNppUZRbmjVK-VcYxU1AgoPKkJ7ruT27X9AwpVsM8F92xux8jKu4Nlm6tfOYyUCsuGSvxLRD1damLw5GpeRewYjXCVfK7ylDwCHhyisaaII0IRIdw=="
  }
}
```

我们直接使用 `host`字段就可以

这里需要注意 链接webSocket需要进行拼接 在B站直播间进行`F12`查看

![image-20220424125640854](https://cdn.jsdelivr.net/gh/duogongneng/OneMyBlogImg@master/image-20220424125640854.png)

最后我们的WebSocket链接拼接成

```
wss://broadcastlv.chat.bilibili.com/sub
```

然后我们就可以进行连接了 我们还是写个方法 然后在这个方法里面进行处理

```js
function openSocket(url, room_id) {
  let ws = new WebSocket(`wss://${url}/sub`);
  let json = {
    uid: 0, // 0表示未登录
    roomid: room_id, //上面获取到的room_id
    protover: 1,
    platform: "web",
    clientver: "1.4.0",
  };
  // WebSocket连接成功回调
  ws.onopen = function () {
    console.log("WebSocket 已连接上");
    ws.send(getCertification(JSON.stringify(json)).buffer);
    //组合认证数据包 并发送
  };


  // WebSocket连接关闭回调
  ws.onclose = function () {
    console.log("连接已关闭");
  };
}
```

**注意点：**

- json 数据 这是我们需要提交进行认证使用的否则是连接不上的
- 这里用到一个方法下面会进行讲解 因为b站接受的数据需要进行封包处理才能连接
- 还需要一个奖字符串转 bytes

#### 封包格式

封包由头部和数据组成，**字节序均为大端模式**

头部格式：

| 偏移量 | 长度 | 含义                  |
| ------ | ---- | --------------------- |
| 0      | 4    | 封包总大小            |
| 4      | 2    | 头部长度              |
| 6      | 2    | 协议版本，目前是1     |
| 8      | 4    | 操作码（封包类型）    |
| 12     | 4    | sequence，可以取常数1 |

已知的操作码：

| 操作码 | 含义                              |
| ------ | --------------------------------- |
| 2      | 客户端发送的心跳包                |
| 3      | 人气值，数据不是JSON，是4字节整数 |
| 5      | 命令，数据中`['cmd']`表示具体命令 |
| 7      | 认证并加入房间                    |
| 8      | 服务器发送的心跳包                |

数据格式：一般为JSON字符串UTF-8编码

```js
//组合认证数据包
function getCertification(json) {
  var bytes = str2bytes(json); //字符串转bytes
  var n1 = new ArrayBuffer(bytes.length + 16);
  var i = new DataView(n1);
  i.setUint32(0, bytes.length + 16), //封包总大小
    i.setUint16(4, 16), //头部长度
    i.setUint16(6, 1), //协议版本
    i.setUint32(8, 7), //操作码 7表示认证并加入房间
    i.setUint32(12, 1); //就1
  for (var r = 0; r < bytes.length; r++) {
    i.setUint8(16 + r, bytes[r]); //把要认证的数据添加进去
  }
  return i; //返回
}
```

**JavaScript DataView**

DataView提供了用于在ArrayBuffer中读取和写入多种数字类型的低级接口。

让我们看看JavaScript的列表< strong> DataView 方法及其说明。

| 方法                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| DataView.getFloat32() | DataView.getFloat32()方法用于在指定位置获取32位浮点数。      |
| DataView.getFloat64() | DataView.getFloat64()方法用于在指定位置获取64位float(double)数字。 |
| DataView.getInt16()   | DataView.getInt16()方法用于在指定位置获取带符号的16位整数(短)数字。 |
| Dataview.getInt32()   | DataView.getInt32()方法用于在指定位置获取带符号的32位整数(长整数)。 |
| DataView.getInt8()    | DataView.getInt8()方法用于在指定位置获取带符号的8位整数(字节)数字。 |
| DataView.getUint16()  | DataView.getUint16()方法用于在指定位置获取无符号的16位整数(无符号的短整数)。 |
| DataView.getUint32()  | DataView.getUint32()方法用于在指定位置获取无符号的32位整数(无符号长整数)。 |
| DataView.getUint8()   | DataView.getUint8()方法用于在指定位置获取无符号的8位整数(无符号字节)数字。 |

```js
//字符串转bytes //这个方法是从网上找的QAQ
function str2bytes(str) {
  const bytes = [];
  let c;
  const len = str.length;
  for (let i = 0; i < len; i++) {
    c = str.charCodeAt(i);
    if (c >= 0x010000 && c <= 0x10ffff) {
      bytes.push(((c >> 18) & 0x07) | 0xf0);
      bytes.push(((c >> 12) & 0x3f) | 0x80);
      bytes.push(((c >> 6) & 0x3f) | 0x80);
      bytes.push((c & 0x3f) | 0x80);
    } else if (c >= 0x000800 && c <= 0x00ffff) {
      bytes.push(((c >> 12) & 0x0f) | 0xe0);
      bytes.push(((c >> 6) & 0x3f) | 0x80);
      bytes.push((c & 0x3f) | 0x80);
    } else if (c >= 0x000080 && c <= 0x0007ff) {
      bytes.push(((c >> 6) & 0x1f) | 0xc0);
      bytes.push((c & 0x3f) | 0x80);
    } else {
      bytes.push(c & 0xff);
    }
  }
  return bytes;
}
```

上面这两个方法是进行认证使用也就是进行连接这一步 就算完成了

链接上不算完成 B站的心跳是30秒一次也就是

我们可以在 连接的时候写个定时器 每30秒进行连接一次 然后在连接关闭的时候进行 清除定时器

我们来改造 `openSocket` 方法

```js
function openSocket(url, room_id) {
	let timer = null
  let ws = new WebSocket(`wss://${url}/sub`);
  let json = {
    uid: 0, // 0表示未登录
    roomid: room_id, //上面获取到的room_id
    protover: 1,
    platform: "web",
    clientver: "1.4.0",
  };
  // WebSocket连接成功回调
  ws.onopen = function () {
    console.log("WebSocket 已连接上");
    ws.send(getCertification(JSON.stringify(json)).buffer);
    //组合认证数据包 并发送
    
    //心跳包的定时器
    timer = setInterval(function () {
      //定时器 注意声明timer变量
      var n1 = new ArrayBuffer(16);
      var i = new DataView(n1);
      i.setUint32(0, 0), //封包总大小
        i.setUint16(4, 16), //头部长度
        i.setUint16(6, 1), //协议版本
        i.setUint32(8, 2), // 操作码 2 心跳包
        i.setUint32(12, 1); //就1
      ws.send(i.buffer); //发送
    }, 30000);
  };


  // WebSocket连接关闭回调
  ws.onclose = function () {
    console.log("连接已关闭");
    if (timer != null) clearInterval(timer);
  };
}
```

这连接就算真正的完事了

那么接下来就是获取消息了

这个应该是放在 `openSocket` 方法里面的

```js
//WebSocket接收数据回调
ws.onmessage = function (evt) {
  var blob = evt.data;
  //对数据进行解码 decode方法
  decode(blob, function (packet) {

  }
 );
};
```

我们可以看下控制台数据

![image-20220424132518466](https://cdn.jsdelivr.net/gh/duogongneng/OneMyBlogImg@master/image-20220424132518466.png)

这里可以看到又一些数据是没有进行压缩的 可以直接进行使用的

但是还有一些数据是进行压缩了的

![image-20220424132603712](https://cdn.jsdelivr.net/gh/duogongneng/OneMyBlogImg@master/image-20220424132603712.png)

我们就可以对这两种情况进行分析

我们创建`decode`函数

```js
/**
 * blob blob数据
 * call 回调 解析数据会通过回调返回数据
 */
function decode(blob, call) {
  // 文本解码器
	var textDecoder = new TextDecoder("utf-8");
  let reader = new FileReader();
  reader.onload = function (e) {
    let buffer = new Uint8Array(e.target.result);
    let result = {};
    result.packetLen = readInt(buffer, 0, 4);
    result.headerLen = readInt(buffer, 4, 2);
    result.ver = readInt(buffer, 6, 2);
    result.op = readInt(buffer, 8, 4);
    result.seq = readInt(buffer, 12, 4);
    if (result.op == 5) {
      result.body = [];
      let offset = 0;
      while (offset < buffer.length) {
        let packetLen = readInt(buffer, offset + 0, 4);
        let headerLen = 16; // readInt(buffer,offset + 4,4)
        let data = buffer.slice(offset + headerLen, offset + packetLen);

        let body = "{}";
        if (result.ver == 2) {
          //协议版本为 2 时  数据有进行压缩 通过pako.js 进行解压
          body = textDecoder.decode(pako.inflate(data));
        } else {
          //协议版本为 0 时  数据没有进行压缩
          body = textDecoder.decode(data);
        }
        if (body) {
          // 同一条消息中可能存在多条信息，用正则筛出来
          // eslint-disable-next-line no-control-regex
          const group = body.split(/[\x00-\x1f]+/);
          group.forEach((item) => {
            try {
              result.body.push(JSON.parse(item));
            } catch (e) {
              // 忽略非JSON字符串，通常情况下为分隔符
            }
          });
        }
        offset += packetLen;
      }
    }
    //回调
    call(result);
  };
  reader.readAsArrayBuffer(blob);
}
```

这里使用了一个方法

```js
// 从buffer中读取int
const readInt = function (buffer, start, len) {
  let result = 0;
  for (let i = len - 1; i >= 0; i--) {
    result += Math.pow(256, len - i - 1) * buffer[start + i];
  }
  return result;
};
```

当返回值协议是2的时候我们需要进行解压然后在返回数据

这里我们解压用到一个包是`pako.js` https://www.npmjs.com/package/pako

然后我们继续完善 `ws.onmessage`

```js
//WebSocket接收数据回调
ws.onmessage = function (evt) {
  var blob = evt.data;
  //对数据进行解码 decode方法
  decode(blob, function (packet) {
		//解码成功回调
      if (packet.op == 5) {
        //会同时有多个 数发过来 所以要循环
        for (let i = 0; i < packet.body.length; i++) {
          var element = packet.body[i];
          //做一下简单的打印
          console.log(element); //数据格式从打印中就可以分析出来啦
          //cmd = DANMU_MSG 是弹幕
          if (element.cmd == "DANMU_MSG") {
            console.log(
              "uid: " +
                element.info[2][0] +
                " 用户: " +
                element.info[2][1] +
                " \n内容: " +
                element.info[1]
            );
          }
          //cmd = INTERACT_WORD 有人进入直播了
          else if (element.cmd == "INTERACT_WORD") {
            console.log("进入直播: " + element.data.uname);
          }
          //还有其他的
        }
      }
  }
 );
};
```

#### 命令包

根据前端代码，数据也可能是多条命令的数组，不过我只收到过单条命令。每条命令中`['cmd']`表示具体命令

已知的命令：

| 命令          | 含义             |
| ------------- | ---------------- |
| DANMU_MSG     | 收到弹幕         |
| SEND_GIFT     | 有人送礼         |
| WELCOME       | 欢迎加入房间     |
| WELCOME_GUARD | 欢迎房管加入房间 |
| SYS_MSG       | 系统消息         |
| PREPARING     | 主播准备中       |
| LIVE          | 直播开始         |
| WISH_BOTTLE   | 许愿瓶？         |

