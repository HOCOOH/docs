---
sidebar_position: 17
---
# 使用 Tongsuo 签发 SM2 证书
### SM2 测试证书
也可以直接使用Tongsuo项目里面的自签发SM2证书，_仅用于测试_；
服务端SM2证书，[https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-first-crt.pem](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-first-crt.pem)

服务端SM2私钥，[https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-first-key.pem](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-first-key.pem)

CA证书，[https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-root.crt](https://github.com/Tongsuo-Project/Tongsuo/blob/master/test/certs/sm2-root.crt)

## 一. 生成 sm2 的私钥:
```bash
openssl ecparam -genkey -name SM2 -out sm2.key
```
## 二. 生成证书签名请求 csr
```bash
openssl req -new -key sm2.key -out sm2.csr -sm3 -sigopt "sm2_id:1234567812345678"
```
## 三. 生成证书(两种方式)
### 简易版(一般用于生成自签名测试证书用)
```bash
openssl x509 -req -in sm2.csr  -signkey sm2.key -out sm2.crt -sm3 -sm2-id 1234567812345678 -sigopt "sm2_id:1234567812345678"
```
### 正式版(用于构建正式证书链)
以构建一个三级证书链为例，见`Tongsuo/test_certs/sm2-cert-sign`，通过`gen-sm2-cert-sign-dir.sh`生成签发证书的文件目录 1. 编写`openssl.cnf`，见`Tongsuo/test_certs/sm2-cert-sign/ca`目录下 2. 通过步骤一，二生成根证书的私钥和csr 3. 生成自签名根证书：
```bash
openssl ca -selfsign -config openssl.cnf -in csr/sm2-root.csr -extensions v3_ca -days 3650 -out sm2-root.crt
```

1. 通过步骤一，二生成中间证书的私钥和`sm2-intermediate-ca.csr`
2. 生成中间ca
```bash
openssl ca -config openssl.cnf -extensions v3_intermediate_ca -days 3650  -in csr/sm2-intermediate-ca.csr -out sm2-intermediate-ca.crt -sigopt "sm2_id:1234567812345678" -sm2-id "1234567812345678" -md sm3
```

1. 为中间ca编写`openssl_middleca.cnf`，见`BabaSSL/test_certs/sm2-cert`目录下
2. 通过步骤一，二生成叶子证书的私钥和`sm2-leaf.csr`
3. 生成叶子证书
```bash
openssl ca -config openssl_middleca.cnf -extensions server_cert -days 3650  -in csr/sm2-leaf.csr -out sm2-leaf.crt -sigopt "sm2_id:1234567812345678" -sm2-id "1234567812345678" -md sm3
```
## 生成crl(证书吊销)
吊销叶子证书
```bash
openssl ca -revoke certs/sm2-leaf.crt -cert certs/sm2-root.crt -key private/sm2-root.key -config openssl.cnf -md sm3 -sm2-id 1234567812345678 -sigopt "sm2_id:1234567812345678"
```
生成crl
```bash
openssl ca -gencrl -out sm2-leaf.crl -cert certs/sm2-root.crt -key private/sm2-root.key -config openssl.cnf
```
