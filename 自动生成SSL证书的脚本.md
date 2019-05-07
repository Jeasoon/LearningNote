
# 自动生成SSL证书的脚本

[toc]

基于Lnux系统下的`openssl`和jdk`keytool`工具

## 1. 脚本和配置

### 1.1 生成https证书脚本

```bash
#! /bin/bash

FILE_PREFIX=tls
RSA_BITS_NUM=2048
VALID_DAYS=3650

PASS_RSA=jeasoon
PASS_P12=jeasoon
PASS_JKS=jeasoon

CRT_ALIAS=jeasoon
CRT_COUNTRY_NAME=CN
CRT_PROVINCE_NAME=Beijing
CRT_CITY_NAME=Beijing
CRT_ORGANIZATION_NAME=jeasoon
CRT_ORGANIZATION_UNIT_NAME=jeasoon
CRT_DOMAIN=*.jeasoon.com
CRT_EMAIL=jeasoon@jeasoon.com
CRT_EXTRA_CHALLENGE_PASSWD=jeasoon
CRT_EXTRA_OPTINAL_COMPANY_NAME=Jeasoon

# 2.1 生成私钥
echo -e "\n----------------------------------------------------------\n生成私钥\n"
openssl genrsa -des3 -passout pass:$PASS_RSA -out $FILE_PREFIX.pem $RSA_BITS_NUM

# 2.2 除去密码口令
echo -e "\n----------------------------------------------------------\n除去密码口令\n"
openssl rsa -in $FILE_PREFIX.pem -out $FILE_PREFIX.key -passin pass:$PASS_RSA

# 2.3 生成证书请求
echo -e "\n----------------------------------------------------------\n生成证书请求\n"
openssl req -new -days $VALID_DAYS -key $FILE_PREFIX.key -out $FILE_PREFIX.csr << EOF
$CRT_COUNTRY_NAME
$CRT_PROVINCE_NAME
$CRT_CITY_NAME
$CRT_ORGANIZATION_NAME
$CRT_ORGANIZATION_UNIT_NAME
$CRT_DOMAIN
$CRT_EMAIL
$CRT_EXTRA_CHALLENGE_PASSWD
$CRT_EXTRA_OPTINAL_COMPANY_NAME
EOF

# 2.4 生成证书
echo -e "\n\n----------------------------------------------------------\n生成证书\n"
openssl x509 -req -days $VALID_DAYS -signkey $FILE_PREFIX.key -in $FILE_PREFIX.csr -out $FILE_PREFIX.crt

# 2.5 crt转为p12证书
echo -e "\n----------------------------------------------------------\ncrt转为p12证书\n"
openssl pkcs12 -export -in $FILE_PREFIX.crt -inkey $FILE_PREFIX.key -name $CRT_ALIAS -passout pass:$PASS_P12 -out $FILE_PREFIX.p12

# 2.6 p12和jks证书互转
echo -e "\n----------------------------------------------------------\np12和jks证书互转\n"
keytool -importkeystore -srckeystore $FILE_PREFIX.p12 -srcstoretype PKCS12 -deststoretype JKS -srcstorepass $PASS_P12 -deststorepass $PASS_JKS -destkeystore $FILE_PREFIX.jks

# 2.7 证书查看
echo -e "\n----------------------------------------------------------\n证书查看\n"
keytool -list -v -storepass $PASS_JKS -keystore $FILE_PREFIX.jks
echo

```

## 1.2 Tomcat配置

修改`tomcat根目录/conf/server.xml`

```xml
<Connector port="443" protocol="org.apache.coyote.http11.Http11NioProtocol"
           maxThreads="150" SSLEnabled="true" >
    <SSLHostConfig>
        <Certificate certificateKeystoreFile="tls/tls.jks"
                     certificateKeystorePassword="jeasoon"
                     certificateKeyPassword="jeasoon"
                     certificateKeyAlias="jeasoon"
                     type="RSA" />
    </SSLHostConfig>
</Connector>
```

## 2. 脚本步骤详解

### 2.1 生成私钥

```bash
openssl genrsa -des3 -passout pass:密码口令 -out tls.pem 2048
```

选项:

* genrsa: genrsa相关命令
* -des3: 加密方式, encrypt the generated key with DES in ede cbc mode (168 bit key)
* -passout: 输出密码口令, 后接 `pass:密码口令`, 如果省略, 需要手动输入密码口令
* -out: 输出文件路径, 后接文件路径; 生成的tls.pem内容为文本
* 1024/2048: 1024/2048位私钥, numbits

### 2.2 除去密码口令

```bash
openssl rsa -in tls.pem -out tls.key -passin pass:密码口令
```

选项:

* rsa: rsa相关命令
* -in: 第一步生成的pem私钥, 后接私钥文件路径
* -out: 输出文件路径, 后接文件路径; 生成的tls.key内容为文本
* -passin: 输入密码口令, 后接 `pass:密码口令`, 如果省略, 需要手动输入密码口令

### 2.3 生成证书请求

```bash
openssl req -new -days 3650 -key tls.key -out tls.csr
```

选项:

* req: req相关命令
* -new: 生成新的证书签名请求
* -days: 有效天数, 后接数字, 天数
* -key: 私钥key路径, 后接第二步生成的key路径
* -out: 输出文件路径, 后接文件路径; 生成的tls.csr内容为文本

交互输入:

```text
国家代码(可空): Country Name (2 letter code) [AU]:
CN
省份代码(可空): State or Province Name (full name) [Some-State]:
Beijing
城市代码(可空): Locality Name (eg, city) []:
Beijing
公司名称(可空): Organization Name (eg, company) [Internet Widgits Pty Ltd]:
Jeasoon
部门名称(可空): Organizational Unit Name (eg, section) []:
Jeasoon
授权域名: Common Name (e.g. server FQDN or YOUR name) []:
*.jeasoon.com
邮件地址(可空): Email Address []:
jeasoon@jeasoon.com
Extra信息(可空): A challenge password []:
jeasoon
Extra信息(可空): An optional company name []:
jeasoon
```

### 2.4 生成证书

```bash
openssl x509 -req -days 3650 -signkey tls.key -in tls.csr -out tls.crt
```

选项:

* x509: x509相关命令
* -req: 输入一个证书请求, 签名并输出
* -days: 有效天数, 后接数字, 天数
* -signkey: 输入私钥, 后接第二步生成的key文件路径
* -in: 输入csr证书, 后接第三步生成的csr证书请求路径
* -out: 输出文件路径, 后接文件路径; 生成的tls.crt内容为文本

### 2.5 crt转为p12证书

```bash
openssl pkcs12 -export -in tls.crt -inkey tls.key -name vixtel -passout pass:密码口令 -out tls.p12
```

选项:

* pkcs12: pkcs12相关命令
* -export: 导出操作
* -in: 输入证书, 后接第四步生成的crt证书路径
* -days: 有效天数, 后接数字, 天数
* -signkey: 输入私钥, 后接第二步生成的key文件路径
* -name: 别名
* -passout: 输出密码口令, 后接 `pass:密码口令`, 如果省略, 需要手动输入密码口令
* -out: 输出文件路径, 后接文件路径; 生成的tls.p12内容为二进制

### 2.6 p12和jks证书互转

**keytool为jdk工具**

p12 转为 jks:

```bash
keytool -importkeystore -srckeystore tls.p12 -srcstoretype PKCS12 -deststoretype JKS -srcstorepass 输入密码口令 -deststorepass 输出密码口令 -destkeystore tls.jks
```

jks 转为 p12:

```bash
keytool -importkeystore -srckeystore tls.jks -srcstoretype JKS -deststoretype PKCS12 -srcstorepass 输入密码口令 -deststorepass 输出密码口令 -destkeystore tls.p12
```

选项:

* -importkeystore: 导入证书并输出指定证书
* -srckeystore: 输入证书路径, 后跟输入证书路径
* -destkeystore: 输出证书路径, 后跟输出证书路径
* -srcstoretype: 输入证书类型, 后跟输入证书类型, PKCS12/JKS
* -deststoretype: 输出证书类型, 后跟输出证书类型, JKS/PKCS12
* -srcstorepass: 输入密码口令, 后接输入证书的密码口令
* -deststorepass: 输出密码口令, 后接输出证书的密码口令

### 2.7 证书查看

```bash
keytool -list -v -storepass 密码口令 -keystore tls.jks
```

选项:

* -list: 证书更改或查看操作
* -v: 详细输出
* -storepass: 证书密码口令
* -keystore: 证书路径, 后跟要查看的证书路径

交互输出:

```text
Keystore type: jks
Keystore provider: SUN

Your keystore contains 1 entry

Alias name: jeasoon
Creation date: Oct 30, 2018
Entry type: PrivateKeyEntry
Certificate chain length: 1
Certificate[1]:
Owner: EMAILADDRESS=jeasoon@jeasoon.com, CN=*.jeasoon.com, O=Vixtel, L=Beijing, ST=Beijing, C=CN
Issuer: EMAILADDRESS=jeasoon@jeasoon.com, CN=*.jeasoon.com, O=Vixtel, L=Beijing, ST=Beijing, C=CN
Serial number: d561956d9762442a
Valid from: Tue Oct 30 17:18:37 CST 2018 until: Fri Oct 27 17:18:37 CST 2028
Certificate fingerprints:
         MD5:  28:82:FC:F2:05:DB:BD:7F:9A:30:3B:DC:92:0A:AF:BE
         SHA1: 62:EC:7A:D5:7A:4B:1C:67:A9:04:FD:8B:B7:4C:5E:9F:D4:7B:0A:8F
         SHA256: 15:20:DA:E2:0D:07:05:99:4F:5F:9C:AA:CF:8F:B3:68:E2:79:27:52:2E:34:52:7C:D5:F6:0E:5E:55:A6:5B:0F
Signature algorithm name: SHA256withRSA
Subject Public Key Algorithm: 2048-bit RSA key
Version: 1


*******************************************
*******************************************


```
