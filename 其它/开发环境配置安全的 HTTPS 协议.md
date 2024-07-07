**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**

<hr />

# 简介

本文采用实战 + 原理的方式来讲述，目的是为了在开发环境使用安全的 https 协议，并了解其背后涉及的原理。

其中的**安全**是指浏览器不报安全提醒，比如：

![image-20240703110132468](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031101493.png)

在上图所示中，可以通过点击`继续前往`从而忽略浏览器的提示，但我们知道，在陌生网站上这么操作就意味着你离被攻击不远了。但我更希望是这样的效果：

![image-20240703110206868](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031102903.png)

**不安全的 https 有什么问题？？**

上图可以通过点击`继续前往`来解决，但有一些场景你绕不过去，比如：

* service worker 场景
  * service worker 只能在安全的 https 协议下注册，但 localhost 除外，localhost 上下文被认为是安全的
  * service worker 中的 fetch 请求的目标 server 如果是不安全的 https 服务，请求也会失败

* 浏览器的 AJAX 请求场景，比如通过 fetch API，请求不安全的 server 会失败，比如 qiankun 主应用加载不安全的子应用，需要先在浏览器中访问子应用，让浏览器忽略子应用的不安全，qiankun 才能正常加载

# 开发环境如何配置安全的 https 协议

照例，先上结论，后讲原理。主要分为三步：

* 生成自签名证书
* 将证书加入系统根证书列表，并选择始终信任
* 项目（https server）集成证书 + 私钥

## 生成自签名证书

假设以下操作均在`/Users/liyongning/studyspace/ssl`目录完成

新建 openssl.cnf 文件，内容如下

```cnf[req]
[req]
prompt = no
default_bits = 4096
default_md = sha512
distinguished_name = dn
x509_extensions = v3_req

[dn]
C=CN
ST=BeiJing
L=BeiJing
O=Development
OU=Fe
CN=self-signed-certificate
emailAddress=test@example.com

[v3_req]
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName=@alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = dev.online.360.cn
IP.1 = 127.0.0.1
IP.2 = 0.0.0.0
```

其中有几个需要注意的字段

* [alt_names] 下的配置，表示证书的生效范围，即哪些域名或 IP
* 通过 DNS.x 来配置域名，假设你的证书想在多个域名上生效，你就分别配置多个 DNS.x 即可，比如 DNS.1 和 DNS.2
* IP.x 和 DNS.x 同理，只不过是针对 IP 地址的
* CN 字段的值是证书放到`系统 —— 证书`中的名字，比如
  * ![image-20240703113251567](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031132650.png)

大家使用时只需更改 DNS.x 或 IP.x 即可。

接下来，执行如下命令生成密钥和证书文件

```shell
openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -config openssl.cnf  -keyout key.pem -out cert.pem
```

效果如下，其中 key.pem 是私钥，cert.pem 是证书

![image-20240703113731945](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031137008.png)

## 信任根证书

将上一步生成的 cert.pem 放到根证书列表中，操作如下

### Mac

* Command + 空格，搜索`钥匙串访问`，并打开

* 找到系统钥匙串 -> 系统 -> 证书

* 将 cert.pem 文件拖进来，在列表中能看到一条名为 self-signed-certificate 的记录

* 右键这条记录 -> 显示简介，得到如下界面，将`使用此证书时`的选项更换为`始终信任`，然后关闭当前窗口（**一定要关闭，关闭才能保存**）


![](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031150152.png)

上述所有 Mac 步骤也可以通过如下命令直接搞定：

```shell
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain cert.pem
```

### Windows

Cmd 命令行输入

```shell
certutil -addstore -f "Root" "cert.pem 的路径（文件直接拖进去就可以出路径）"
```

也可以采用双击 cert.pem 文件进行安装

## 项目集成

### Vite 项目

配置 vite.config.js

```javascript
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import fs from 'fs'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [vue()],
  server: {
    host: 'dev.online.360.cn',
    https: {
      key: fs.readFileSync('/Users/liyongning/studyspace/ssl/key.pem'),
      cert: fs.readFileSync('/Users/liyongning/studyspace/ssl/cert.pem')
    }
  }
})
```

效果如下：

![](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031458792.png)

### Webpack 项目

以 vue-cli 为例，配置 vue.config.js

```javascript
const { defineConfig } = require('@vue/cli-service');
const fs = require('fs')

module.exports = defineConfig({ 
  devServer: {
    https: {
      key: fs.readFileSync('/Users/liyongning/studyspace/ssl/key.pem'),
      cert: fs.readFileSync('/Users/liyongning/studyspace/ssl/cert.pem')
    }
  }
})
```

效果如下

![image-20240703150523738](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031505796.png)

### Node 项目

```javascript
const express = require('express');
const https = require('https');
const fs = require('fs');
const path = require('path');

const app = express();

const sslOptions = {
  key: fs.readFileSync(path.resolve('/Users/liyongning/studyspace/ssl', 'key.pem')),
  cert: fs.readFileSync(path.resolve('/Users/liyongning/studyspace/ssl', 'cert.pem'))
};

app.use((_, res, __) => {
  res.send('hello https')
})

https.createServer(sslOptions, app).listen(8083, () => {
  console.log('HTTPS server is running on port 8083');
});
```

效果如下

![image-20240703151305920](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407031513965.png)

至此，自签名证书制作和实际应用都讲完了，接下来的内容就是其中涉及的原理了。

# HTTPS 发展史

http 协议最开始的设计目的很简单，就是传输超文本文件（HTML），没有什么加密的需求，所有数据都是明文传输，这就意味数据在传输过程的每个环节都可能被窃取和篡改。

这个情况严重制约了业务的发展，比如在线购物、支付转账，毫无安全可言，于是在协议栈中引入了安全层，负责数据的加解密，让数据传输更安全。

![image-20240707084153314](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407070841354.png)

可以看到，HTTPS 并不是一个新的协议，只是在 HTTP 层和 TCP 之间加入了一层加解密的逻辑，以前是 HTTP 直接和 TCP 通信，现在是 HTTP -> TSL（SSL 早已废弃）-> TCP，安全层负责对经过的数据进行加解密。所以，HTTPS 的发展史就是安全层逻辑的迭代史。

## 对称加密

![image-20240707084805498](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407070848545.png)

对称加密，就是客户端和服务端使用同一个密钥来进行加解密。那么，问题来了，这个密钥怎么来？答案是：由服务端和客户端使用相同的加密方法、key（随机数）计算得到，在正式传输数据之前，浏览器和服务器之间会有一个协商加解密方式的过程，这个过程就是 HTTPS 建立安全连接的过程：

* 浏览器带着自己支持的加密方案列表和一个随机数，请求服务器
* 服务器收到之后，选择其中的一个加密方法，比如 A，并且也生成一个随机数，返回给浏览器
* 然后浏览器和服务器之间分别相互确认收到了对方的消息，这时浏览器和服务器就有了相同的加密方法和两个随机数
* 于是，浏览器和服务器使用 **加密方法 A + client random + server random** 生成密钥
* 然后，双发利用得到的密钥对传输的数据进行加解密

但这时你会发现，虽然，最终的数据全是通过密钥加密传输的，但是生成密钥时使用的加密方法、随机数都是明文传输的，中间人使用相同的方法和随机数同样可以生成相同的密钥，进而解密 -> 窃取、篡改、伪造 -> 加密数据，所以，一样不安全。

![image-20240707100356551](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407071003582.png)

## 非对称加密

![image-20240707100723557](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407071007602.png)

非对称加密和对称加密相对，分为私钥和公钥两个密钥，加解密使用的是两个不同的密钥，公钥加密的数据只能使用私钥解密，私钥加密的数据同样只能使用公钥解密。

* 浏览器携带着自己支持的加密方法列表向服务器发起请求
* 服务器从加密列表中选择一个加密方法，比如 A，并加上自己生成的公钥一起返回给浏览器

> 服务器会生成一对密钥（公钥 + 私钥），私钥留在自己手里（绝不可泄漏），公钥分发给各个客户端

* 浏览器和服务器相互确认收到信息，从而建立连接
* 浏览器使用公钥加密数据，然后将数据发送给服务端，服务端使用私钥解密数据
* 服务器使用私钥加密数据，然后将数据发送给浏览器，浏览器使用公钥解密数据

接下来，我们分析下整个流程中的数据安全性：

* 由于公钥加密的数据只能由私钥解密，所以，浏览器发送给服务器的数据，是安全的
* 由于公钥是服务器通过明文的方式发放给浏览器，所以，和对称加密存在一样的问题，中间人一样可以窃取公钥和加密方法，所以，服务器发送给浏览器的数据，中间人同样也可以获取到，因此，服务器发送给浏览器的数据，不安全
* 另外，非对称加密相比对称加密在性能上会有数量级级别的差异，所以，全程采用非对称加密算法，性能会差很多

所以，性能差还不能绝对安全。

## 非对称加密 + 对称加密

![image-20240707104641474](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407071046522.png)

意思是，同时采用两种加密方式来保证数据的安全性。协商阶段采用非对称加密的方式，数据传输阶段使用对称加密的方式，通过这样的方式来保证数据传输性能和对称密钥的安全性，即前两种方式的结合版。

* 浏览器带着自己支持的对称加密方法列表、非对称加密方法列表和随机数请求服务器
* 服务器给浏览器返回它从两个加密列表中选择的加密方法、服务器生成的随机数，以及服务器的公钥
* 浏览器收到服务器返回后，另外再生成一个随机数 S，并通过公钥加密后，发送给服务器
* 服务器使用私钥解密后得到浏览器发送的随机数 S，并向浏览器返回确认收到信息
* 这时，浏览器和服务器同时拥有 client random、server random、随机数 S，然后使用相同的加密方法，计算得到相同的密钥作为对称密钥。这里相比对称加密方案的变化是，随机数 S 的传递是加密的形式，所以生成的对称密钥只有通信双方知道
* 后续使用对称密钥对传递的数据进行加解密

这个方案的关键在于非对称密钥 + 双随机数的方式，保证了对称密钥的安全，进而解决了数据安全问题。

## 数字证书

通过非对称加密 + 对称加密结合的方式，我们实现了数据的加密传输。但是，这套方案还存在一个漏洞 —— 中间人攻击。

**浏览器访问的服务器，一定是真的目标服务器吗？**比如，你访问 `baidu.com`，如果你的 DNS 解析被人改了，baidu 的流量被解析到了攻击者的服务器上，上述整体流程就变成了：浏览器 -> 黑客服务器 -> 目标服务器，这三者之间的通信了，即浏览器和目标服务器之间的所有通信都是通过黑客服务器来完成中转的，包括随机数和密钥的传输，即：

* 浏览器看到的 baidu 是黑客的服务器
* 服务器看到的浏览器是黑客的服务器

那有什么方案，可以让浏览器知道自己现在访问的服务器是真正的目标服务器吗？这里我们需要引入一个第三方机构。

以买房为例，你怎么证明你现在居住的房子是你自己的房子？我们可以通过房管局发放的**房产证**来证明。这里引入具有权威的房管局，以及房管局颁发的房产证，大众信任房管局，所以当你出示房产证的时候，就可以证明这个房子确实是你的房子。

同理，在 HTTP 通信的过程里，我们引入权威机构 —— **CA**（Certificate Authority），CA 颁发的证书就是**数字证书**。

![image-20240707154108961](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407071541047.png)

相较于上一个方案，这里引入了数字证书的概念，数字证书有两个作用：

* 让浏览器有机会验证自己访问的服务器是正确的目标服务器，而不是冒名顶起的黑客服务器
* 证书中包含了服务提供商的基础信息以及公钥，所以这套方案中没有专门下发公钥

可以看到，流程中多了一步验证数字证书的流程。

### 数字证书的申请

* 公司准备一对私钥和公钥（私钥绝对保密）
* 公司将自己的公钥、公司信息、站点信息提交给 CA 机构
* CA 机构验证公司和站点信息的有效性，验证通过后
* CA 对公司提交的信息做 hash 运算得到信息摘要，并通过 CA 自己的私钥（CA 机构也有一对自己的非对称密钥）对摘要加密，得到一串数字签名，这个签名会放到证书上，然后将证书颁发给公司
* 公司将证书部署到自己的服务器上

### 浏览器数字证书

浏览器通过验证数字证书，从而得到自己访问的服务器是否为正确的目标服务器。

浏览器从服务器拿到数字证书后：

* 对数字证书上的相关信息做 hash 运算，得到信息摘要
* 通过 CA 机构的公钥解密证书上的数字签名，得到信息摘要
* 对比两个信息摘要，如果内容一致，则说明当前访问的服务器是目标服务器

这里有个问题，**CA 机构的公钥怎么来的？**这个证书有两种获取途径，一是从目标服务器返回，另一个是从网络下载（你从浏览器的请求列表中看不到对应的请求）。

### 会不会有人冒充 CA 机构 ？？

首先，答案是不会。逻辑是这样的：

CA 结构分为根 CA 和 中间 CA，大家都是去中间 CA 去申请证书，而根 CA 是负责给中间 CA 做认证的，而这些中间 CA 又可以去给其他 CA 做认证，这些每个根 CA 都维护了一棵 CA 树，比如：

![image-20240707165627772](https://raw.githubusercontent.com/liyongning/picture-bed/master/liyongning/202407071656856.png)

admin.saas.360 的证书是由 WoTrus DV Server CA 颁发的，而这个中间 CA 又是由 USERTrust RSA Certification Authority 来负责认证的。

所以，这里有一个证书链，浏览器会首先验证 admin.saas.360.cn 的证书是否合法，如果验证通过，再去验 WoTrus DV Server CA 的合法性，如果也合法，再去验证 USERTrust RSA Certification Authority 根 CA 的合法性。

那么问题来，最终的**根 CA 合法性如何验证？**很简单，只需要判断这个根证书在不在操作系统中即可，这些根证书是内置在每个操作系统中的，如果系统中存在，则认为合法，否则就是不合法的。

相信读到这里，你就能理解本文 **开发环境配置安全的 HTTPS 协议** 的背后原理了。

如果某个机构想要成为根 CA，并让它的根证书内置到操作系统中，那么这个机构首先要通过 WebTrust 国际安全审计认证。

## HTTPS 就绝对安全吗？？

这是 HTTPS 认证流程面试题结束时必问的一个问题。答案是：否，HTTPS 不是绝对安全。

相信，读完本文，你就能很清晰的说出来其中的不安全点了：

* 用户自身行为造成的不安全，浏览器会负责验证站点的安全性，如果有风险，会给用户安全提醒，但用户可以选择忽略并继续访问
* 如果用户设备本身遭遇攻击，将非法的证书内置到用户的操作系统中，就像本文开发环境的配置一样，那浏览器也检测不出来



好了，到这里就结束了，相信阅读完本文之后：

* 学到了如何在开发环境配置安全的 HTTPS 协议
* 明白了 HTTPS 安全传输背后的原理，也明白了 HTTPS 并非绝对安全

# 总结

本文从实用角度开始讲，讲了在日常的开发环境中如何配置安全的 HTTPS 协议，即浏览器中没有安全提示

* 一条命令生成自签名证书
* 在系统根证书列表信任自签名证明
* 然后在项目中集成

接下来讲了 HTTPS 的发展史，对称加密、非对称加密、非对称加密 + 对称加密，再到数字证书，从而理解其中的整个原理。

最后讲了一道面试题：**HTTPS 就绝对安全吗？**从而验证对原理的理解程度。

<hr />

**当学习成为了习惯，知识也就变成了常识。** 感谢各位的 **关注**、**点赞**、**收藏**和**评论**。

新视频和文章会第一时间在微信公众号发送，欢迎关注：[李永宁lyn](https://gitee.com/liyongning/typora-image-bed/raw/master/202202171742614.jpg)

文章已收录到 [github 仓库 liyongning/blog](https://github.com/liyongning/blog)，欢迎 Watch 和 Star。

**[更多精彩内容](https://github.com/liyongning/blog/blob/main/README.md)**
