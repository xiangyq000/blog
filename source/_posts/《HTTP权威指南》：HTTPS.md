---
title: '《HTTP权威指南》: HTTPS'
date: 2016-07-28 17:14:38
tags: HTTP
categories: [读书笔记, Network]
---
### 1. X.509证书
#### 1.1 证书格式：
1. 证书版本号(Version)
  版本号指明 X.509 证书的格式版本，现在的值可以为:
  1. v1
  2. v2
  3. v3

2. 证书序列号(Serial Number)
  序列号指定由CA分配给证书的唯一的”数字型标识符”。当证书被取消时，实际上是将此证书的序列号放入由CA签发的CRL中，这也是序列号唯一的原因。
  <!--more-->

3. 签名算法标识符(Signature Algorithm)
  签名算法标识用来指定由CA签发证书时所使用的”签名算法”。算法标识符用来指定CA签发证书时所使用的:
  - 公开密钥算法
    - hash算法
      example: sha256WithRSAEncryption 
      须向国际知名标准组织(如ISO)注册

4. 签发机构名(Issuer)
  此域用来标识签发证书的CA的X.500 DN(DN-Distinguished Name)名字。包括:
  1. 国家(C)
  2. 省市(ST)
  3. 地区(L)
  4. 组织机构(O)
  5. 单位部门(OU)
  6. 通用名(CN)
  7. 邮箱地址

5. 有效期(Validity)
  指定证书的有效期，包括:
  - 证书开始生效的日期时间
  - 证书失效的日期和时间
    每次使用证书时，需要检查证书是否在有效期内。

6. 证书用户名(Subject)
  指定证书持有者的X.500唯一名字。包括:
  1. 国家(C)
  2. 省市(ST)
  3. 地区(L)
  4. 组织机构(O)
  5. 单位部门(OU)
  6. 通用名(CN)
  7. 邮箱地址

7. 证书持有者公开密钥信息(Subject Public Key Info)
  证书持有者公开密钥信息域包含两个重要信息:
  - 证书持有者的公开密钥的值
  - 公开密钥使用的算法标识符。此标识符包含公开密钥算法和hash算法。
8. 扩展项(extension)
  X.509 V3证书是在v2的基础上一标准形式或普通形式增加了扩展项，以使证书能够附带额外信息。标准扩展是指
  由X.509 V3版本定义的对V2版本增加的具有广泛应用前景的扩展项，任何人都可以向一些权威机构，如ISO，来
  注册一些其他扩展，如果这些扩展项应用广泛，也许以后会成为标准扩展项。

9. 签发者唯一标识符(Issuer Unique Identifier)
  签发者唯一标识符在第2版加入证书定义中。此域用在当同一个X.500名字用于多个认证机构时，用一比特字符串来唯一标识签发者的X.500名字。可选。

10. 证书持有者唯一标识符(Subject Unique Identifier)
  持有证书者唯一标识符在第2版的标准中加入X.509证书定义。此域用在当同一个X.500名字用于多个证书持有者时，用一比特字符串来唯一标识证书持有者的X.500名字。可选。

11. 签名算法(Signature Algorithm)
   证书签发机构对证书上述内容的签名算法。
   example:sha256WithRSAEncryption

12. 签名值(Issuer’s Signature)
   证书签发机构对证书上述内容的签名值

#### 1.2 证书样例
```
Data:
  Version:  v3
  Serial Number: 0x1
  Signature Algorithm: SHA1withRSA - 1.2.840.113549.1.1.5
  Issuer: CN=Certificate Manager,OU=netscape,O=ExampleCorp,L=MV,ST=CA,C=US
  Validity: 
    Not Before: Friday, February 21, 2005 12:00:00 AM PST America/Los_Angeles
    Not  After: Monday, February 21, 2007 12:00:00 AM PST America/Los_Angeles
  Subject: CN=Certificate Manager,OU=netscape,O=ExampleCorp,L=MV,ST=CA,C=US
  Subject Public Key Info: 
    Algorithm: RSA - 1.2.840.113549.1.1.1
    Public Key: 
      Exponent: 65537
      Public Key Modulus: (2048 bits) :
        E4:71:2A:CE:E4:24:DC:C4:AB:DF:A3:2E:80:42:0B:D9:
        CF:90:BE:88:4A:5C:C5:B3:73:BF:49:4D:77:31:8A:88:
        15:A7:56:5F:E4:93:68:83:00:BB:4F:C0:47:03:67:F1:
        30:79:43:08:1C:28:A8:97:70:40:CA:64:FA:9E:42:DF:
        35:3D:0E:75:C6:B9:F2:47:0B:D5:CE:24:DD:0A:F7:84:
        4E:FA:16:29:3B:91:D3:EE:24:E9:AF:F6:A1:49:E1:96:
        70:DE:6F:B2:BE:3A:07:1A:0B:FD:FE:2F:75:FD:F9:FC:
        63:69:36:B6:5B:09:C6:84:92:17:9C:3E:64:C3:C4:C9
  Extensions: 
    Identifier: Netscape Certificate Type - 2.16.840.1.113730.1.1
      Critical: no 
      Certificate Usage: 
        SSL CA 
        Secure Email CA 
        ObjectSigning CA 
    Identifier: Basic Constraints - 2.5.29.19
      Critical: yes 
      Is CA: yes 
      Path Length Constraint: UNLIMITED
    Identifier: Subject Key Identifier - 2.5.29.14
      Critical: no 
      Key Identifier: 
        3B:46:83:85:27:BC:F5:9D:8E:63:E3:BE:79:EF:AF:79:
        9C:37:85:84
    Identifier: Authority Key Identifier - 2.5.29.35
      Critical: no 
      Key Identifier: 
        3B:46:83:85:27:BC:F5:9D:8E:63:E3:BE:79:EF:AF:79:
        9C:37:85:84
    Identifier: Key Usage: - 2.5.29.15
      Critical: yes 
      Key Usage: 
        Digital Signature 
        Key CertSign 
        Crl Sign 
  Signature: 
    Algorithm: SHA1withRSA - 1.2.840.113549.1.1.5
    Signature: 
      AA:96:65:3D:10:FA:C7:0B:74:38:2D:93:54:32:C0:5B:
      2F:18:93:E9:7C:32:E6:A4:4F:4E:38:93:61:83:3A:6A:
      A2:11:91:C2:D2:A3:48:07:6C:07:54:A8:B8:42:0E:B4:
      E4:AE:42:B4:B5:36:24:46:4F:83:61:64:13:69:03:DF:
      41:88:0B:CB:39:57:8C:6B:9F:52:7E:26:F9:24:5E:E7:
      BC:FB:FD:93:13:AF:24:3A:8F:DB:E3:DC:C9:F9:1F:67:
      A8:BD:0B:95:84:9D:EB:FC:02:95:A0:49:2C:05:D4:B0:
      35:EA:A6:80:30:20:FF:B1:85:C8:4B:74:D9:DC:BB:50
```

浏览器收到证书时会对签名颁发机构进行检查。若该机构是权威的公共签名机构，浏览器可能已经知道其公开密钥（浏览器会预装很多签名颁发机构的证书），这样就可以验证签名了。

如果对签名颁发机构一无所知，浏览器就无法确定是否应该信任它，这时通常会向用户显示一个对话框，看看他是否相信这个签名发布者（可能是本地的IT部门或软件厂商）。

## 2. HTTPS 连接的建立
在未加密 HTTP 中，客户端会打开一条到 Web 服务器端口 80 的 TCP 连接，发送一条请求报文，接收一条响应报文，关闭连接。

由于 SSL 安全层的存在，HTTPS 中这个过程会略微复杂一些。在 HTTPS 中，客户端首先打开一条到 Web 服务器端口 443（安全 HTTP 的默认端口）的连接。一旦建立了 TCP 连接，客户端和服务器就会初始化 SSL 层，对加密参数进行沟通，并交换密钥。握手完成之后，SSL 初始化就完成了，客户端就可以将请求报文发送给安全层了。在将这些报文发送给 TCP 之前，要先对其进行加密。

在发送已加密的 HTTP 报文之前,客户端和服务器要进行一次 SSL 握手，在这个握手过程中，它们要完成以下工作：

- 交换协议版本号;
- 选择一个两端都了解的密码;
- 对两端的身份进行认证;
- 生成临时的会话密钥，以便加密信道。 