---
layout:         page
title:         【Go】Localhost支持HTTPS之二：Go实现
subtitle:       
date:           2019-04-21
author:         翼
header-img: image/bg.jpg
catalog: true
tags:
---

根据第一篇讲述PKI原理，这里开始实践：用Go实现一个HTTP服务并支持HTTPS。

## 一：创建根证书
我们先创建一个RSA-2048秘钥并且保存为rootCA.key。这个秘钥文件会被用来生成根SSL证书。每次使用这个秘钥文件时OpenSSL都会提示我们输入密码：
```javascript

G:\learn\go\src\Go-Web>openssl
OpenSSL> genrsa -des3 -out rootCA.key 2048
Generating RSA private key, 2048 bit long modulus
..................................................+++
........+++
e is 65537 (0x10001)
Enter pass phrase for rootCA.key:(这里设置密码)
Verifying - Enter pass phrase for rootCA.key:（这里确认密码）
OpenSSL>

```

现在我们可以用上的秘钥文件rootCA.key来创建一个新的根SSL证书，保存为rootCA.pem：
```javascript

OpenSSL> req -x509 -new -nodes -key rootCA.key -sha256 -days 1024 -out rootCA.pem
Enter pass phrase for rootCA.key:(这里输入验证密码)
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:TT
State or Province Name (full name) [Some-State]:TT
Locality Name (eg, city) []:TT
Organization Name (eg, company) [Internet Widgits Pty Ltd]:LocalCorp
Organizational Unit Name (eg, section) []:LocalDep
Common Name (e.g. server FQDN or YOUR name) []:Localhost
Email Address []:localhost@test.com

```
openssl会提示我们输入证书相关信息，比如国家、公司名、邮件等等。

以下内容不明白的话，可以去看第一篇《Localhost支持HTTPS之一：SSL证书信任模型》。

现在这个根证书可以被用来颁发（签名）一个特定的用于本地开发环境的证书。根据第一篇《Localhost支持HTTPS之一：SSL证书信任模型》介绍的内容，过程如下。
## 二：创建网站的终端用户SSL证书

#### 第一步: 创建SSL证书签名申请（certificate signing request）
```javascript

OpenSSL> req -new -sha256 -nodes -out server.csr -newkey rsa:2048 -keyout server.key
Generating a 2048 bit RSA private key
.+++
.............+++
writing new private key to 'server.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:TT
State or Province Name (full name) [Some-State]:TT
Locality Name (eg, city) []:TT
Organization Name (eg, company) [Internet Widgits Pty Ltd]:TestLocal
Organizational Unit Name (eg, section) []:TestLocal
Common Name (e.g. server FQDN or YOUR name) []:TestLocal
Email Address []:test@test.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:********
An optional company name []:TestLocal2
OpenSSL>

```
这里我们创建了一个证书签名申请文件server.csr以及一个私钥server.key。

#### 第二步：创建SSL证书
现在可以用前面的根证书来发布这个证书签名申请，生成我们的终端用户证书文件server.crt：
```javascript

OpenSSL> x509 -req -in server.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out server.crt -days 500 -sha256
Signature ok
subject=/C=TT/ST=TT/L=TT/O=TestLocal/OU=TestLocal/CN=TestLocal/emailAddress=test@test.com
Getting CA Private Key
Enter pass phrase for rootCA.key:(这里输入创建根证书时设置的密码)

```
至此，我们总共创建了如下文件：
```javascript

rootCA.key
rootCA.pem
rootCA.srl
server.crt
server.csr
server.key

```
其中，rootCA.pem是根证书，rootCA.key是根证书私钥，server.crt是网站用的终端用户证书，server.key是终端用户证书私钥。有了这些，加上第一篇介绍的SSL/TLS信任模型，该怎么支持本地HTTPS已经很简单明了了。

## 三、把根SSL证书rootCA.pem添加到本地信任证书中心
这里只说windows上的操作方式。
启动mmc命令，点击在“文件”菜单的“添加/删除管理单元”子菜单添加一个“证书”管理单元。随后就能在左侧“控制台根节点”下面看到“证书”一栏。在下面找到“受信任的根证书颁发机构”下面的“证书”，把我们的根证书rootCA.pem导入进去。至此，本地环境就会信任由此根证书颁发的任何SSL证书，包括我们用于本地网站调试的server.crt。

## 本地站点使用终端用户SSL证书
```go

router := mux.NewRouter()
router.HandleFunc("/", Handler).Methods("GET")
err := http.ListenAndServeTLS("0.0.0.0:443", "server.crt", "server.key", router)
if err != nil {
    es := err.Error()
    fmt.Println(es)
    logger := log.New(os.Stdout, "[server]: ", log.Llongfile)
    logger.Println("ListenAndServeTLS failed: " + err.Error() + es)
}

```

启动服务就可在浏览器里面访问https://localhost来验证了。

这里顺便说一下，我启动服务器出现如下错误：
```go

listen tcp 127.0.0.1:443: bind: An attempt was made to access a socket in a way forbidden by its access permissions.

```
这个错误是由于本地443端口被占用导致的。可以用如下命令查看占用端口的进程信息：
```javascript

netstat -ano | grep 0.0.0.0:443

// 这条命令可以直接列出占用443端口的进程名称
tasklist | grep (netstat -ano | grep 0.0.0.0:443 | awk '{print $NF}') | awk '{print $1}'

```