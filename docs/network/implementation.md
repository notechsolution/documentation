# 100行代码实现一个多功能的代理服务

> TL;DR: 上一篇文章我们介绍了关于计算机网络代理的定义，代理的类型 以及其对应的实现原理。这篇文章将通过手写代码的方式，用100行代码实现一个同时支持HTTP普通代理，隧道代理，以及TCP代理的代理软件。同时介绍中业务代码如何使用HTTP代理。

## 实现HTTP普通代理

《HTTP权威指南》图描述了普通代理的基本原理：

![](.\image\simple-proxy.png)

看样子我们在中间实现一个API服务，满足左手交右手的功能即可。那中间的这个API服务要怎么知道你最终要去的目标服务器是哪里呢？所以这里在客户端跟代理服务器之间要有一个约定，客户端要把目标地址放到url里面，当代理服务器收到请求时，从url解析出目标服务器的地址，再向目标服务器发起请求，把返回的内容再发送给客户端，整个过程就完成了。

下面的20行左右的代码就可以在NodeJS里面实现了这个 普通代理。

```javascript
function request(srcReq, srcRes) {
    const url = new URL(srcReq.url);
    const options = {
        hostname: url.hostname,
        port: url.port || 80,
        path: url.pathname,
        method: srcReq.method,
        headers: srcReq.headers
    }
    // timestamp clientAddress requestMethod URI result/statusCodes responseTime
    const startAt = process.hrtime();
    let logLine = `${Date.now()} ${requestIP.getClientIp(srcReq)} ${srcReq.method} ${srcReq.url}`
    const destRequest = http.request(options, destResponse => {
        srcRes.writeHead(destResponse.statusCode, destResponse.headers);
        destResponse.pipe(srcRes)
        logLine += ` ${destResponse.statusCode} ${_.get(destResponse.headers,"content-type",'-')} ${_getTotalTime(startAt)}`
        access.log(logLine);
    }).on('error', e => {
        srcRes.end();
        logLine += ` ERROR ${_getTotalTime(startAt)}`;
        error.log(logLine + '\r\n' + e)
    })
    srcReq.pipe(destRequest);
}

http.createServer().on('request', request)；
```

代码的详细介绍如下：

1. 代理服务收到请求后，从url解析出目标服务器的地址，同时将客户端请求的 path, method 以及 headers都搬过去 ，代码 2-9行
2. 代理服务向目标服务器发起一个另一个HTTP请求，代码 13-22 行
3. 把客户端请求的内容，用管道的方式 发送给目标服务器，代码23行
4. 把目标服务器返回的内容，同样用管道的方式 发送回给客户端，代码15行
5. 通过Nodejs的 http模块，创建一个服务器。从这里也可以看出来，这个代理服务器是在HTTP 层上面的



## 实现HTTP隧道代理

关于HTTP普通代理的问题可以参考上一篇文章。HTTP 隧道代理的工作原理如下图：

![](.\image\tunnel-proxy.png)

通过HTTP CONNECT方法，做客户端跟代理中间建立起一个隧道，然后代理服务器跟目标服务器发起一个SSL的TCP连接，最后代理将与目标服务器建立TCP连接，与客户端之间建立的TCP连接对接起来，从而实现了这个隧道代理的过程。

下面这另外的20行代码也实现了这一过程：

```javascript
function connect(srcReq, srcSocket) {
    const url = new URL(`http://${srcReq.url}`);
    const startAt = process.hrtime();
    let logLine = `${Date.now()} CONNECT ${url.hostname}:${url.port}`
    const destSocket = net.connect(parseInt(url.port), url.hostname, () => {
        srcSocket.write('HTTP/1.1 200 Connection Established\r\n\r\n');
        destSocket.pipe(srcSocket);
        logLine += ` ${_getTotalTime(startAt)}`
        access.log(logLine);
    }).on('error', e => {
        srcSocket.end();
        logLine += ` ERROR ${_getTotalTime(startAt)}`;
        error.log(logLine + '\r\n' + e)
    })

    srcSocket.pipe(destSocket);
}
http.createServer().on('connect', connect)
```

代码的详细介绍如下：

1. 当代理服务器收到CONNECT请求时，一样的从请求的url里面解析出目标服务器的 hostname跟port
2. 当客户端与代理发起HTTP CONNECT之后，两者之间就建立一个Socket的链接。这个从connect 事件的参数就可以看出两者建立的是socket连接
3. 接着代理向目标服务器发起一个 TCP连接请求
4. 同样的，代理把客户端请求的内容通过管道pipe给目标服务器，把目标服务器返回的内容通过管道会送给客户端。要注意的是：这里双方对接的是TCP层面的请求，而不是HTTP层面的请求，这样才能处理HTTPS的证书验证问题，以及隐私保护。

## 实现TCP代理

有些服务之间的链接是直接在TCP层面的，比如一个OSI第五层的协议，那他们之间如果需要用到代理的话，用第七层的HTTP代理需要兜兜转转，而且需要遮遮掩掩，因为这违背了OSI的原则。所以我们可以实现一个TCP层面的代理。

下面的代码也实现另一个TCP的代理：

```javascript
function connection(socket) {
    const startAt = process.hrtime();
    let logLine = `${Date.now()} ${socket.remoteAddress} ${socket.remotePort} ${socket.remoteFamily}`;
    access.log(logLine);
    socket.setTimeout(SOCKET_TIMEOUT, () => {
        errorLog.log(`${logLine} Socket Timeout ${SOCKET_TIMEOUT}`);
    })
    const destSocket = net.connect(parseInt(destPort), destHost, () => {
        destSocket.pipe(socket);
    }).on('error', e => {
        socket.end();
    });
    socket.pipe(destSocket);
    socket.on('timeout', function () {
        logLine += ` ${_getTotalTime(startAt)}`
        errorLog.log(logLine);
        socket.end('Timed out!');
    });
    socket.on('error',function(error){
        errorLog.log('Error : ' + error);
    });
}
net.createServer().server.on('connection', connection);
```

TCP代理的实现过程跟 隧道代理很像，都是在TCP层面直接建立连接。但唯一不同的是 它不是通过HTTP CONNECT的方法来打开这个链接，而是直接就在TCP层面就透传过去。Nginx的stream实现的就是这一过程。

这里只是为了演示的效果，TCP代理的目标服务器是通过配置读取的，真实实现过程当中，可以从配置文件中读取，这样可以实现多个目标服务器的透传。比如像Nginx的stream的配置如下：

```nginx
stream {
    server {
        listen     127.0.0.1:12345;
        proxy_pass backend.example.com:12345;
        proxy_bind 127.0.0.1:12345;
    }
}
```



## 多功能代理服务

我们把上面三种不同的代理揉在一起，就可以用100行实现一个简化版的，支持多重不同协议的多功能代理服务：

```javascript
const http = require('http');
const net = require('net');
const {URL} = require('url');
const requestIP = require('request-ip');
const _ = require('lodash');
const accessLog = console;
const errorLog = console;
const SOCKET_TIMEOUT = process.env.SOCKET_TIMEOUT || 60000;
const destHost = process.env.DEST_HOST || '';
const destPort = process.env.DEST_PORT || 0;

function request(srcReq, srcRes) {
    const url = new URL(srcReq.url);
    const options = {
        hostname: url.hostname,
        port: url.port || 80,
        path: url.pathname,
        method: srcReq.method,
        headers: srcReq.headers
    }
    // timestamp clientAddress requestMethod URI result/statusCodes responseTime
    const startAt = process.hrtime();
    let logLine = `${Date.now()} ${requestIP.getClientIp(srcReq)} ${srcReq.method} ${srcReq.url}`
    const destRequest = http.request(options, destResponse => {
        srcRes.writeHead(destResponse.statusCode, destResponse.headers);
        destResponse.pipe(srcRes)
        logLine += ` ${destResponse.statusCode} ${_.get(destResponse.headers,"content-type",'-')} ${_getTotalTime(startAt)}`
        accessLog.log(logLine);
    }).on('error', e => {
        srcRes.end();
        logLine += ` ERROR ${_getTotalTime(startAt)}`;
        errorLog.log(logLine + '\r\n' + e)
    })
    srcReq.pipe(destRequest);
}

function _getTotalTime(startAt){
    // time elapsed from request start
    const elapsed = process.hrtime(startAt)
    const ms = (elapsed[0] * 1e3) + (elapsed[1] * 1e-6)
    return ms.toFixed(3)
}
function connect(srcReq, srcSocket) {
    const url = new URL(`http://${srcReq.url}`);
    const startAt = process.hrtime();
    let logLine = `${Date.now()} CONNECT ${url.hostname}:${url.port}`
    const destSocket = net.connect(parseInt(url.port), url.hostname, () => {
        srcSocket.write('HTTP/1.1 200 Connection Established\r\n\r\n');
        destSocket.pipe(srcSocket);
        logLine += ` ${_getTotalTime(startAt)}`
        accessLog.log(logLine);
    }).on('error', e => {
        srcSocket.end();
        logLine += ` ERROR ${_getTotalTime(startAt)}`;
        errorLog.log(logLine + '\r\n' + e)
    })
    srcSocket.pipe(destSocket);
}

function connection(socket) {
    const startAt = process.hrtime();
    let logLine = `${Date.now()} ${socket.remoteAddress} ${socket.remotePort} ${socket.remoteFamily}`;
    accessLog.log(logLine);
    socket.setTimeout(SOCKET_TIMEOUT, () => {
        errorLog.log(`${logLine} Socket Timeout ${SOCKET_TIMEOUT}`);
    })
    const destSocket = net.connect(parseInt(destPort), destHost, () => {
        destSocket.pipe(socket);
    }).on('error', e => {
        socket.end();
    });
    socket.pipe(destSocket);
    socket.on('timeout', function () {
        logLine += ` ${_getTotalTime(startAt)}`
        errorLog.log(logLine);
        socket.end('Timed out!');
    });
    socket.on('error',function(error){
        errorLog.log('Error : ' + error);
    });
}

const httpPort = parseInt(process.env.PORT) || 8888;
console.log(`Starting My Proxy...`)
http.createServer()
    .on('request', request)
    .on('connect', connect)
    .listen(httpPort, '0.0.0.0', null, () => {
        console.log(`Started My Proxy at port ${httpPort}.`)
    });

const tcpPort = parseInt(process.env.TCP_PORT) || 33555;
net.createServer()
    .on('listening', () => console.log('Server is listening'))
    .on('connection', connection)
    .on('error', e => console.log(`Error found ${e}`))
    .listen(tcpPort, () => {
    console.log('Server is listening at port ' + tcpPort);
})

```





## 代码中如何使用HTTP 代理

有了HTTP代理之后，我们这代码里应该怎样写才能使请求通过代理服务器发出去。最简单也最常用的就是通过 proxy agent的组件来实现，比如通过https proxy agent这样的包来让代码通过代理服务器发出请求：

```javascript
var url = require('url');
var https = require('https');
var HttpsProxyAgent = require('https-proxy-agent');
 
// HTTP/HTTPS proxy to connect to
var proxy = process.env.http_proxy || 'http://168.63.76.32:3128';
console.log('using proxy server %j', proxy);
 
// HTTPS endpoint for the proxy to connect to
var endpoint = process.argv[2] || 'https://graph.facebook.com/tootallnate';
console.log('attempting to GET %j', endpoint);
var options = url.parse(endpoint);
 
// create an instance of the `HttpsProxyAgent` class with the proxy server information
var agent = new HttpsProxyAgent(proxy);
options.agent = agent;
 
https.get(options, function (res) {
  console.log('"response" event!', res.headers);
  res.pipe(process.stdout);
});
```

当然，如果想看更仔细一点，自己发起一个HTTP CONNECT的连接也是可以的：

```javascript
const targetHost = 'www.baidu.com';
const req = http.request({
    host: 'localhost',
    port: 8888,
    method: 'CONNECT',
    path: `${targetHost}:443`,
});

req.on('connect', function (res, socket, head) {
    const tlsSocket = Tls.connect({
        host:targetHost,
        socket: socket
    }, function () {
        tlsSocket.write(`GET / HTTP/1.1\r\nHost: ${targetHost}:443+\r\nConnection: close\r\n\r\n`);
    });

    tlsSocket.on('secureConnect', function (data) {
        console.log(`[event]-secureConnect: ${tlsSocket.authorized}, ${tlsSocket.authorizationError}`);
    })
    tlsSocket.on('data', function (data) {
        console.log(data.toString());
    });
    tlsSocket.on('error', function (error) {
        console.error(error);
    });
});

req.end();
```





