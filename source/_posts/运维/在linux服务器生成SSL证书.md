---
title: 在linux服务器生成SSL证书
categories:
  - 运维
  - Linux
tags:
  - 运维
  - Linux
  - SSL
abbrlink: c6ea9d8
date: 2025-02-06 15:05:00
---

``` powershell
# 1.创建服务器证书密钥文件 server.key：
$ openssl genrsa -des3 -out server.key 2048
# 输入密码，确认密码，自己随便定义，但是要记住，后面会用到。
# 2.创建服务器证书的申请文件 server.csr
$ openssl req -new -key server.key -out server.csr
# 输出内容为：
Enter pass phrase for root.key: ← 输入前面创建的密码
Country Name (2 letter code) [AU]:CN ← 国家代号，中国输入CN
State or Province Name (full name) [Some-State]:BeiJing ← 省的全名，拼音
Locality Name (eg, city) []:BeiJing ← 市的全名，拼音
Organization Name (eg, company) [Internet Widgits Pty Ltd]:MyCompany Corp. ← 公司英文名
Organizational Unit Name (eg, section) []: ← 可以不输入
Common Name (eg, YOUR name) []: ← 输入域名，如：iot.conet.com
Email Address []:admin@mycompany.com ← 电子邮箱，可随意填
Please enter the following ‘extra’ attributes
to be sent with your certificate request
A challenge password []: ← 可以不输入
An optional company name []: ← 可以不输入
# 4.备份一份服务器密钥文件
$ cp server.key server.key.org
# 5.去除文件口令
$ openssl rsa -in server.key.org -out server.key
# 6.生成证书文件server.crt，365=一年
$ openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```