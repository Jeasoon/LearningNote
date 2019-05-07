
# SSL证书理解及生成

[toc]

### 1. x509证书

509证书一般会用到三类文件, key, csr, crt

* key: 私用密钥, openssl格式, 通常是rsa算法
* csr: 证书请求文件, **用于申请/制作证书**. 在制作csr文件的时候, 必须使用自己的私钥来签署申请, 还可以设定一个密钥
* crt: CA认证后的证书文件, 签署人用自己的key给你签署的凭证

### 2. 证书分类

* 根证书: CA自签名证书, 通常为x509, 由CA私钥和CA证书请求文件生成
* 服务器证书: 服务器自己生成csr证书请求文件, 通过CA证书和CA私钥生成
* 客户端证书: 客户端自己生成csr证书请求文件, 通过CA证书和CA私钥生成

### 3. 证书生成

#### 3.1 生成CA证书

> 主要生成命令:

```bash
# 生成ca私钥
openssl genrsa -out ca.key 4096

# 生成ca证书请求文件
openssl req -new -key ca.key -out ca.csr

# 生成ca证书(自签名)
openssl x509 -req -days 36500 -signkey ca.key -in ca.csr -out ca.crt
```

> 当前目录生成文件有:

```bash
-rw-r--r-- 1 root root 1766 Apr 17 21:45 ca.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 ca.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 ca.crt
```

#### 3.2 生成服务器证书

> 主要生成命令有:

```bash
# 生成server私钥
openssl genrsa -out server.key 1024

# 生成server证书请求文件
openssl req -new -key server.key -out server.csr

# 生成server证书(ca证书和ca私钥签名)
openssl ca -keyfile ca.key -cert ca.crt -in server.csr -out server.crt
```

> 当前目录生成文件有:

```bash
-rw-r--r-- 1 root root 1766 Apr 17 21:45 server.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 server.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 server.crt
```

#### 3.3 生成客户端证书(同服务器, 单向认证不需要)

> 主要生成命令有:

```bash
# 生成client私钥
openssl genrsa -out client.key 1024

# 生成client证书请求文件
openssl req -new -key client.key -out client.csr

# 生成client证书(ca证书和ca私钥签名)
openssl ca -keyfile ca.key -cert ca.crt -in client.csr -out client.crt
```

> 当前目录生成文件有:

```bash
-rw-r--r-- 1 root root 1766 Apr 17 21:45 client.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 client.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 client.crt
```

#### 3.4 生成pem格式证书(不一定使用到)

> 主要生成命令有:

```bash
# 生成ca pem
cat ca.crt ca.key > ca.pem

# 生成server端pem
cat server.crt server.key > server.pem

# 生成client端pem
cat client.crt client.key > client.pem
```

#### 3.5 查看生成内容

> 生成的文件:

```bash
-rw-r--r-- 1 root root 1766 Apr 17 21:45 ca.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 ca.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 ca.crt
-rw-r--r-- 1 root root 4856 Apr 17 21:53 ca.pem

-rw-r--r-- 1 root root 1766 Apr 17 21:45 server.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 server.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 server.crt
-rw-r--r-- 1 root root 4856 Apr 17 21:53 server.pem

-rw-r--r-- 1 root root 1766 Apr 17 21:45 client.key
-rw-r--r-- 1 root root 1013 Apr 17 21:46 client.csr
-rw-r--r-- 1 root root 4856 Apr 17 21:53 client.crt
-rw-r--r-- 1 root root 4856 Apr 17 21:53 server.pem
```

> 查看证书:

```bash
openssl x509 -in server.crt -noout -text
...
```

#### 3.6 可能遇到的问题

> 在使用ca证书和私钥给服务器端或者客户端签名时可能出现下面错误:

```bash
$ openssl ca -keyfile ca.key -cert ca.crt -in server.csr -out server.crt

Using configuration from /usr/lib/ssl/openssl.cnf
Enter pass phrase for ca.key:
I am unable to access the ./demoCA/newcerts directory
./demoCA/newcerts: No such file or directory
```

主要是`/usr/lib/ssl/openssl.cnf`配置问题, 可在当前目录做如下操作:

```bash
# 根据错误提示创建demoCA/newcerts目录
mkdir -p demoCA/newcerts

# 创建序列号文件, 第一行为01, 第二行为空行
echo -e "01\n" > demoCA/serial

# 创建index.txt文件
touch demoCA/index.txt
```
