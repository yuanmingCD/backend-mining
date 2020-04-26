# SSL

[TOC]

## 简介

### 什么是SSL

- SSL：Secure Socket Layer，安全套接字层，它位于TCP层与Application层之间。提供对Application数据的加密保护（密文），完整性保护（不被篡改）等安全服务，它缺省工作在TCP 443 端口，一般对HTTP加密，即俗称HTTPS。
- TLS：Transport Layer Secure，更关注的是提供安全的传输服务，它很灵活，如果可能，它可以工作在TCP，也可以UDP （DTLS），也可以工作在数据链路层，比如802.1x EAP-TLS。

处于ISO七层网络模型中的位置，传输层（四层）之上，应用层（七层）之下

### SSL可以用来做什么

- **加密** 防止数据被窃听
- **鉴权** 防止身份被冒充
- **完整性校验** 防止数据被篡改

#### 加密

| 加密方式   | 密钥         | 安全性 | 计算量 | 实现复杂度 | 常用算法                                                     |
| ---------- | ------------ | ------ | ------ | ---------- | ------------------------------------------------------------ |
| 对称加密   | 相同密钥     | 低     | 小     | 易         | DES、RC                                                      |
| 非对称加密 | 公钥和私钥对 | 高     | 大     | 难         | [RSA](https://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html) |

结合对称密钥以及非对称密钥优缺点，在SSL中应用大致过程为，通信双方在首先在握手阶段通过rsa加密通信协商生成"对话密钥"，采用"对话密钥"进行加密通信。

#### 鉴权

就要有以下三种校验模式

- 不校验
- 单向校验
- 双向校验

##### 不校验

客户端与服务端互相不校验身份，不使用校验功能

##### 单向校验

客户端会校验服务端合法性，服务端不会去校验客户端，比如通过浏览器访问某一个网站。

##### 双向校验

客户端和服务端会互相校验合法性，用于客户端与服务端安全通信的程序。

#### 完整性校验

摘要/哈希

可以把不定长度的数据映射成固定长度，相同内容经过摘要可以得到相同的结果，稍有改动，得到的摘要结果就会不同，但是也会有小概率的哈希冲突问题。摘要不同于加密算法，近似于一个单向的过程，很难从摘要反推原信息，常用的摘要算法有MD5、SHA1、SHA256、SHA512等。

数字签名

简而言之，即摘要加密，客户端将摘要用私钥加密，然后连同原报文一起发送给服务端，服务端使用约定的摘要算法对原报文进行摘要计算，与客户端传过来的摘要进行对比验签。一致可以认为数据没有被篡改。

### 文件种类

- **公钥**

与私钥成对存在，可以传播，服务端与客户端建立握手的过程中会发送给客户端，客户端用其来加密数据。

- **私钥**

与公钥成对存在，保存在服务端，不能泄露，用来解密公钥加密的数据

- **CA**

CA(Certificate Authority，证书授权中心)，是一个单位，来管理发放数字证书的。由它发放的证书就叫 CA 证书，以区别于个人使用工具随意生成的数字证书，查看 CA 证书，里面有两项重要内容，一个是颂发给谁，另一个是由谁颂发的。

- **证书**

证书主要是**用来做身份校验**，包含电子签证机关的信息、公钥用户信息、公钥、签名和有效期。其中

- 公钥：这里的公钥服务端的公钥，
- 签名：用hash散列函数计算公开的明文信息的信息摘要，然后采用CA的私钥对信息摘要进行加密，加密完的密文就是签名

总结来说，证书 = 公钥 + 签名 +申请者和颁发者的信息

（1）自签名证书(一般用于顶级证书、根证书): 证书的名称和认证机构的名称相同.
（2）根证书：根证书是CA认证中心给自己颁发的证书,是信任链的起始点。任何安装CA根证书的服务器都意味着对这个CA认证中心是信任的。

证书的签发

### 文件格式

实际上，术语X.509证书通常指的是IETF的PKIX证书和X.509 v3证书标准的CRL 文件，即如RFC 5280（通常称为PKIX for Public Key Infrastructure（X.509））中规定的。

为适应不同的应用场景，SSL所用到的文件格式衍生了很多，几种常用证书类型对比如下：

| 后缀      | 格式         | 内容         | 密码保护 |
| --------- | ------------ | ------------ | -------- |
| .DER .CER | 二进制       | 仅证书       |          |
| .PEM      | 文本         | 证书和私钥   |          |
| .CRT      | 二进制或文本 | 仅证书       |          |
| .PFX .P12 | 二进制       | 证书和私钥   | 有       |
| .JKS      | 二进制       | 证书和私钥   | 有       |
| .KEY      | 二进制或文本 | 公钥或者私钥 |          |

#### 格式对比

- DER

该格式是二进制文件内容，Java 和 Windows 服务器偏向于使用这种编码格式。

```sh
# genrsa 生成私钥和公钥
openssl genrsa -out key.pem 2048 
# 生成证书

# 查看证书内容
openssl x509 -in certificate.der -inform der -text -noout
```

- PEM

Privacy Enhanced Mail，一般为文本格式，以 `-----BEGIN...` 开头，以 `-----END...` 结尾。中间的内容是 BASE64 编码。这种格式可以保存证书和私钥，有时我们也把PEM 格式的私钥的后缀改为 .key 以区别证书与私钥。具体你可以看文件的内容。

中间为base64 encodeString，根据注释信息有几种不同的格式，参考https://kb.kutu66.com/java/post_787598

```
1.证书内容
-----BEGIN CERTIFICATE-----
... ...
-----END CERTIFICATE-----

2. 未加密的PKCS#8编码数据
-----BEGIN PRIVATE KEY-----
... ...
-----END PRIVATE KEY-----

3.PKCS#1
-----BEGIN RSA PRIVATE KEY-----
... ...
-----END RSA PRIVATE KEY-----

4.PKCS#12格式
可通过`openssl pkcs8 -in serverCerts.pem -topk8 -nocrypt -out serverCerts_key.pem`转化
-----BEGIN ENCRYPTED PRIVATE KEY-----
... ...
-----END ENCRYPTED PRIVATE KEY-----

```

```shell
# genrsa 生成私钥和公钥
openssl genrsa -out key.pem 2048 
# 查看内容
openssl x509 -in certificate.pem -text -noout
```

- CRT

Certificate 的简称，有可能是 PEM 编码格式，也有可能是 DER 编码格式。

- PFX

Predecessor of PKCS#12，这种格式是二进制格式，且证书和私钥存在一个 PFX 文件中。一般用于 Windows 上的 IIS 服务器。改格式的文件一般会有一个密码用于保证私钥的安全。

```shell
openssl pkcs12 -in cert.pfx
```

- JKS

Java Key Storage， JAVA 的专属格式，利用 JDK 的 `keytool` 的工具可以进行格式转换。

```shell
keytool -v -list -keystore  ******.jks
```

- KEY

```shell
# 查看内容
openssl rsa -in mykey.key -text -noout
```

#### 格式之间的转化

DER to PEM

```shell
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

PEM to DER 

```shell
openssl x509 -in cert.crt -outform der -out cert.der
```

PFX/P12 to PEM

```shell
openssl pkcs12 -in serverCerts.pfx -out serverCerts.pem -nodes
openssl pkcs12 -in serverCerts.p12 -out serverCerts.pem
```

JKS to P12

```shell
keytool -importkeystore -srckeystore serverCerts.jks \
   -destkeystore serverCerts.p12 \
   -srcstoretype jks \
   -deststoretype pkcs12
```

PEM(encrypted) to PEM(nocrypt)

```shell
openssl pkcs8 -in serverCerts.pem -topk8 -nocrypt -out serverCerts_key.pem
```



## 原理

- 握手
- 数据传输

### 握手过程

主要分为四个通信过程

- client hello
  - 支持的协议版本，比如TLS 1.0版。
  - 客户端生成的随机数 Random1，用于生成对话密钥
  - 客户端支持的加密套件（Support Ciphers）
  - 客户端支持的压缩算法
- ServerHello
  - 服务端生成的随机数 Random2，用于生成对话密钥
  - ServerCertificate 服务端证书
  - ServerKeyExchange 确认使用的加密证书
  - ServerHelloDone
- ClientCertificate
  - 客户端生成的随机数 Random1，又称"pre-master key"，用于生成对话密钥
  - CertificateVerify
  - ClientKeyExchange
  - ChangeCipherSpec 通知服务端改用加密通信
  - EncryptedHandshakeMessage 第一条加密数据，加密数据校验
- Server
  - ChangeCipherSpec 切换到加密
  - EncryptedHandshakeMessage

为什么采用三个随机数,握手阶段还未加密，因而整个过程的数据可以被窃取到，由于SSL协议中证书是静态的，因此十分有必要引入一种随机因素来保证协商出来的密钥的随机性，这样生成的密钥才不会每次都一样。

#### 单向校验

在第二阶段，客户端获取到服务端证书之后进行校验

#### 双向校验

在第二阶段，客户端获取到服务端证书之后进行校验

在第三阶段，服务端获取到客户端证书之后进行校验



### 通信过程



抓包分析可以参考[SSL/TLS 握手过程详解](https://www.jianshu.com/p/7158568e4867)这篇文章

### 优化

SSL进行了一些优化以提高性能

- 会话复用（Session Ticket）
- 抢跑（False Start）



更多参考[TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake.html)

## 实战

>  Talk is cheap，show me the code

双向校验开发

```shell
# 创建服务端秘钥
keytool -genkey -alias server -keysize 1024 -validity 3650 -keyalg RSA -dname "CN=localhost" -keypass serverPass -storepass serverPass -keystore certs/serverCerts.jks

# 导出服务端秘钥
keytool -export -alias server -keystore certs/serverCerts.jks -storepass serverPass -file certs/serverCert.cer

# 创建客户端秘钥
keytool -genkey -alias client -keysize 1024 -validity 3650 -keyalg RSA -dname "CN=PF,OU=YJC,O=YJC,L=BJ,S=BJ,C=ZN" -keypass clientPass -storepass clientPass -keystore certs/clientCerts.jks

# 导出客户端秘钥
keytool -export -alias client -keystore certs/clientCerts.jks -file certs/clientCert.cer -storepass clientPass

# 将客户端的证书导入到服务端的信任证书仓库中
keytool -import -trustcacerts -alias smcc -file certs/clientCert.cer -storepass serverPass -keystore certs/serverCerts.jks

# 将服务端的证书导入到客户端的信任证书仓库中
keytool -import -trustcacerts -alias smccClient -file certs/serverCert.cer -storepass clientPass -keystore certs/clientCerts.jks
```

注意：不同版本的JDK默认的证书不一样



参考文章

[SSL/TLS 握手过程详解](https://www.jianshu.com/p/7158568e4867)

[TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake.html)