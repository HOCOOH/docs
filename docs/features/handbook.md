---
sidebar_position: 3
---
# 铜锁探“密”训练营实验手册

铜锁探“密”活动由开放原子开源基金会和铜锁社区共同举办，包含5次课程，以“抽丝剥茧，循序渐进，一起揭开商用密码的面纱”为主题，旨在让参与者更加深入地了解商用密码的原理和应用方法。
录播链接如下：

1. 商用密码介绍和铜锁密码库入门，[https://live.csdn.net/room/csdnnews/NJPxluNk](https://live.csdn.net/room/csdnnews/NJPxluNk)
2. 常用国密算法编程入门与实战，[https://live.csdn.net/room/csdnnews/hZel8pLF](https://live.csdn.net/room/csdnnews/hZel8pLF)
3. 实战国密证书和国密传输协议，[https://live.csdn.net/room/csdnnews/8fs5oVfT](https://live.csdn.net/room/csdnnews/8fs5oVfT)
4. 实战铜锁国密应用和结营作业说明，[https://live.csdn.net/room/csdnnews/BzeD1mGt](https://live.csdn.net/room/csdnnews/BzeD1mGt)
5. 训练营结营&作品点评，[https://live.csdn.net/room/csdnnews/fsZDshwb](https://live.csdn.net/room/csdnnews/fsZDshwb)

## 实验环境说明

实验手册中大部分实验以docker环境为主，基于Ubuntu 20.04容器镜像。或者直接运行于Ubuntu 20.04操作系统或虚拟机也可以。比如电脑使用的Windows系统，可以通过安装Docker环境、Linux虚拟机、或者Linux子系统进行实验。其他Linux系统或者macOS系统可以参考本实验内容。

安装docker：

可以参考docker官方安装教程，详见[https://docs.docker.com/engine/install/](https://docs.docker.com/engine/install/)。

创建docker容器：
```bash
docker run -d -it --name tongsuolab ubuntu:20.04 bash
```
执行所有实验，都需要先进入docker容器。进入docker容器：
```bash
docker exec -it tongsuolab bash
```
其他软件，直接在操作系统上安装即可。

- 支持国密协议的浏览器，比如360企业安全浏览器。
- Wireshark，用于抓包分析国密协议。

## 构建铜锁密码库

基于铜锁密码库源代码进行构建和安装。代码下载地址：

AtomGit地址（推荐）：[https://atomgit.com/tongsuo/Tongsuo](https://atomgit.com/)

GitHub地址：[https://github.com/Tongsuo-Project/Tongsuo](https://atomgit.com/tongsuo/Tongsuo)

下载代码需要使用git工具；构建铜锁密码库需要使用perl、gcc、make等基础开发工具。

安装依赖：
```bash
# 需要先更新软件包索引
apt update

apt install git gcc make -y
```

进入Shell终端执行以下命令：
```bash
git clone https://github.com/Tongsuo-Project/Tongsuo

cd Tongsuo

./config --prefix=/opt/tongsuo enable-ntls enable-ssl-trace -Wl,-rpath,/opt/tongsuo/lib64 --debug
make -j
make install

```
查看安装情况：
```bash
ls -l /opt/tongsuo
```
![image.png](img/handbook1.png)

产看铜锁版本，执行如下命令：
```bash
/opt/tongsuo/bin/tongsuo version
```
![image.png](img/handbook2.png)

## SM2&SM3&SM4算法实战

### 实战SM4加解密算法

```bash
echo "hello tongsuo" > msg.bin

# SM4-CBC加密
/opt/tongsuo/bin/tongsuo enc -K "3f342e9d67d6ce7be701756af7bac8f2" -e -sm4-cbc -in msg.bin -iv "1fb2d42fb36e2e88a220b04f2e49aa13" -nosalt -out cipher.bin

# SM4-CBC解密
/opt/tongsuo/bin/tongsuo enc -K "3f342e9d67d6ce7be701756af7bac8f2" -d -sm4-cbc -in cipher.bin -iv "1fb2d42fb36e2e88a220b04f2e49aa13" -nosalt -out msg2.bin

# 比较解密的明文和原来的消息是否一样
diff msg.bin msg2.bin
```
### 实战SM3杂凑算法

```bash
echo -n "hello tongsuo" | /opt/tongsuo/bin/tongsuo dgst -sm3
```
结果如下：
![image.png](img/handbook3.png)

### 实战SM2签名和验签

```bash
# 生成一个随机内容文件
dd if=/dev/urandom of=msg.bin bs=1024 count=1

# SM2私钥签名，签名算法为SM2withSM3，Tongsuo/test/certs/sm2.key来自Tongsuo源代码仓库
/opt/tongsuo/bin/tongsuo dgst -sm3 -sign Tongsuo/test/certs/sm2.key -out sigfile msg.bin

# SM2公钥验签，Tongsuo/test/certs/sm2pub.key来自Tongsuo源代码仓库
/opt/tongsuo/bin/tongsuo dgst -sm3 -verify Tongsuo/test/certs/sm2pub.key -signature sigfile msg.bin

```
签名正确时，验证成功可以看到：
![image.png](img/handbook4.png)

## SM2&SM3&SM4算法编程入门

### SM4加解密算法编程入门

SM4加密
```c
#include <openssl/evp.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm4_enc(const unsigned char *key, const unsigned char *iv,
                   unsigned char *in, int inlen, unsigned char *out,
                   int *outlen)
{
    EVP_CIPHER_CTX *ctx = NULL;
    int outl, tmplen;

    if ((ctx = EVP_CIPHER_CTX_new()) == NULL
        || !EVP_EncryptInit_ex(ctx, EVP_sm4_cbc(), NULL, key, iv)
        || !EVP_EncryptUpdate(ctx, out, &outl, in, inlen)
        || !EVP_EncryptFinal_ex(ctx, out + outl, &tmplen)) {
        EVP_CIPHER_CTX_free(ctx);
        return 0;
    }

    EVP_CIPHER_CTX_free(ctx);

    if (outlen)
        *outlen = outl + tmplen;

    return 1;
}

int main()
{
    unsigned char key[] = {
        0x3f, 0x34, 0x2e, 0x9d, 0x67, 0xd6, 0xce, 0x7b,
        0xe7, 0x01, 0x75, 0x6a, 0xf7, 0xba, 0xc8, 0xf2,};
    unsigned char iv[] = {
        0x1f, 0xb2, 0xd4, 0x2f, 0xb3, 0x6e, 0x2e, 0x88,
        0xa2, 0x20, 0xb0, 0x4f, 0x2e, 0x49, 0xaa, 0x13,};
    unsigned char in[] = "hello tongsuo";
    unsigned char *out = NULL;
    int outlen;
    int ret;

    out = malloc(sizeof(in) + EVP_MAX_BLOCK_LENGTH);
    assert(out != NULL);

    ret = sm4_enc(key, iv, in, strlen(in), out, &outlen);
    assert(ret == 1);

    for (size_t i = 0; i < outlen; i++)
        printf("%x", out[i]);

    printf("\n");

    free(out);

    return 0;
}

```
将内容保存为sm4_enc.c，并编译运行：
```c
gcc sm4_enc.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
输出明文消息的密文如下：
![image.png](img/handbook5.png)

SM4解密：
```c
#include <openssl/evp.h>
#include <openssl/err.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm4_dec(const unsigned char *key, const unsigned char *iv,
                   unsigned char *in, int inlen, unsigned char *out,
                   int *outlen)
{
    EVP_CIPHER_CTX *ctx = NULL;
    int outl, tmplen;

    if ((ctx = EVP_CIPHER_CTX_new()) == NULL
        || !EVP_DecryptInit_ex(ctx, EVP_sm4_cbc(), NULL, key, iv)
        || !EVP_DecryptUpdate(ctx, out, &outl, in, inlen)
        || !EVP_DecryptFinal_ex(ctx, out + outl, &tmplen)) {
		ERR_print_errors_fp(stderr);
        EVP_CIPHER_CTX_free(ctx);
        return 0;
    }

    EVP_CIPHER_CTX_free(ctx);

    if (outlen)
        *outlen = outl + tmplen;

    return 1;
}

int main()
{
    unsigned char key[] = {
        0x3f, 0x34, 0x2e, 0x9d, 0x67, 0xd6, 0xce, 0x7b,
        0xe7, 0x01, 0x75, 0x6a, 0xf7, 0xba, 0xc8, 0xf2,};
    unsigned char iv[] = {
        0x1f, 0xb2, 0xd4, 0x2f, 0xb3, 0x6e, 0x2e, 0x88,
        0xa2, 0x20, 0xb0, 0x4f, 0x2e, 0x49, 0xaa, 0x13,};
    unsigned char in[] = {
        0xe2, 0x44, 0xdb, 0xeb, 0x97, 0x58, 0x83, 0x1e,
        0xa8, 0x7b, 0x7c, 0xeb, 0x27, 0x8e, 0x6e, 0x5d,};
    unsigned char *out = NULL;
    int outlen;
    int ret;

    out = malloc(sizeof(in));
    assert(out != NULL);

    ret = sm4_dec(key, iv, in, sizeof(in), out, &outlen);
    assert(ret == 1);

    printf("%*s\n", outlen, out);

    free(out);

    return 0;
}
// gcc sm4_dec.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64
```
保存内容为sm4_dec.c，编译并执行：
```bash
gcc sm4_dec.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
输出如下：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/21453368/1687334065888-58826405-6ff7-41c6-8664-dce8f14df784.png#averageHue=%23111111&clientId=ubb3fd28b-eb33-4&from=paste&height=40&id=u01d12dcf&originHeight=80&originWidth=604&originalType=binary&ratio=2&rotation=0&showTitle=false&size=11794&status=done&style=none&taskId=u6eb5fd88-f97e-44e0-8b56-15a9dbd78a6&title=&width=302)

### SM3杂凑算法编程入门

计算SM3杂凑：
```c
#include <openssl/evp.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm3(const unsigned char *in, size_t inlen, unsigned char *out)
{
    EVP_MD_CTX *mctx = NULL;

    if ((mctx = EVP_MD_CTX_new()) == NULL
        || !EVP_DigestInit_ex(mctx, EVP_sm3(), NULL)
        || !EVP_DigestUpdate(mctx, in, inlen)
        || !EVP_DigestFinal_ex(mctx, out, NULL)) {
        EVP_MD_CTX_free(mctx);
        return 0;
    }

    EVP_MD_CTX_free(mctx);
    return 1;
}

int main()
{
    unsigned char in[] = "hello tongsuo";
    unsigned char out[EVP_MAX_MD_SIZE];
    int ret;

    ret = sm3(in, strlen(in), out);
    assert(ret == 1);

    for (int i = 0; i < EVP_MD_size(EVP_sm3()); i++)
        printf("%x", out[i]);

    printf("\n");
    return 0;
}

```
保存内容为sm3.c，编译运行如下：
```bash
gcc sm3.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
运行结果如下：
![image.png](img/handbook6.png)

### SM2签名算法编程入门

SM2签名：
```c
#include <openssl/evp.h>
#include <openssl/err.h>
#include <openssl/pem.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm2_sign(EVP_PKEY *pkey, const unsigned char *in, size_t inlen,
                    unsigned char *out, size_t *outlen)
{
    EVP_MD_CTX *mctx = NULL;

    if ((mctx = EVP_MD_CTX_new()) == NULL
        || !EVP_DigestSignInit(mctx, NULL, EVP_sm3(), NULL, pkey)
        || !EVP_DigestSign(mctx, out, outlen, in, inlen))
    {
        ERR_print_errors_fp(stderr);
        EVP_MD_CTX_free(mctx);
        return 0;
    }

    EVP_MD_CTX_free(mctx);
    return 1;
}

int main()
{
    unsigned char msg[] = "hello tongsuo";
    unsigned char *sig = NULL;
    EVP_PKEY *pkey = NULL;
    BIO *bio = NULL;
    size_t siglen;
    unsigned char privkey[] =
        "-----BEGIN PRIVATE KEY-----\n"
"MIGHAgEAMBMGByqGSM49AgEGCCqBHM9VAYItBG0wawIBAQQgSKhk+4xGyDI+IS2H\n"
"WVfFPDxh1qv5+wtrddaIsGNXGZihRANCAAQwqeNkWp7fiu1KZnuDkAucpM8piEzE\n"
"TL1ymrcrOBvv8mhNNkeb20asbWgFQI2zOrSM99/sXGn9rM2/usM/Mlca\n"
"-----END PRIVATE KEY-----\n";
    int ret;

    bio = BIO_new_mem_buf(privkey, -1);
    assert(bio != NULL);

    pkey = PEM_read_bio_PrivateKey(bio, NULL, NULL, NULL);
    assert(pkey != NULL);

    siglen = EVP_PKEY_size(pkey);
    sig = malloc(siglen);

    ret = sm2_sign(pkey, msg, strlen((const char *)msg), sig, &siglen);
    assert(ret == 1);

    for (size_t i = 0; i < siglen; i++)
        printf("%02x", sig[i]);

    printf("\n");

    EVP_PKEY_free(pkey);
    BIO_free(bio);
    free(sig);
    return 0;
}

```
保存代码到文件sm2_sign.c，编译并运行
```bash
gcc sm2_sign.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
输出签名结果：
![image.png](img/handbook7.png)

SM2验签：
```c
#include <openssl/evp.h>
#include <openssl/err.h>
#include <openssl/pem.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm2_verify(EVP_PKEY *pkey, const unsigned char *msg, size_t msglen,
                      unsigned char *sig, size_t siglen)
{
    EVP_MD_CTX *mctx = NULL;

    if ((mctx = EVP_MD_CTX_new()) == NULL
        || EVP_DigestVerifyInit(mctx, NULL, EVP_sm3(), NULL, pkey) != 1
        || EVP_DigestVerify(mctx, sig, siglen, msg, msglen) != 1)
    {
        ERR_print_errors_fp(stderr);
        EVP_MD_CTX_free(mctx);
        return 0;
    }

    EVP_MD_CTX_free(mctx);
    return 1;
}

int main()
{
    unsigned char msg[] = "hello tongsuo";
    unsigned char sig[] = {
0x30, 0x44, 0x02, 0x20, 0x64, 0x68, 0xab, 0xee, 0x05, 0xa0, 0x46, 0xef,
0xe2, 0xcd, 0x05, 0x79, 0xa2, 0xa2, 0xe8, 0x9a, 0xf2, 0x70, 0xc4, 0xa3,
0x36, 0x6b, 0xd3, 0x37, 0x2c, 0xee, 0x9a, 0x7f, 0x26, 0x2b, 0x61, 0x01,
0x02, 0x20, 0x73, 0x51, 0x81, 0x60, 0x40, 0xfc, 0x10, 0x32, 0xde, 0xd0,
0x57, 0x4b, 0x43, 0xbb, 0xe8, 0xf0, 0x92, 0x6d, 0x48, 0x24, 0x24, 0x32,
0x6d, 0x1a, 0x52, 0xb2, 0xb0, 0x4e, 0x8a, 0xb5, 0x55, 0x80,};
    EVP_PKEY *pkey = NULL;
    BIO *bio = NULL;
    unsigned char pubkey[] =
        "-----BEGIN PUBLIC KEY-----\n"
"MFkwEwYHKoZIzj0CAQYIKoEcz1UBgi0DQgAEMKnjZFqe34rtSmZ7g5ALnKTPKYhM\n"
"xEy9cpq3Kzgb7/JoTTZHm9tGrG1oBUCNszq0jPff7Fxp/azNv7rDPzJXGg==\n"
"-----END PUBLIC KEY-----\n";
    int ret;

    bio = BIO_new_mem_buf(pubkey, -1);
    assert(bio != NULL);

    pkey = PEM_read_bio_PUBKEY(bio, NULL, NULL, NULL);
    assert(pkey != NULL);

    ret = sm2_verify(pkey, msg, strlen((const char *)msg), sig, sizeof(sig));
    assert(ret == 1);

    EVP_PKEY_free(pkey);
    BIO_free(bio);
    return 0;
}

// gcc sm2_verify.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64
```
保存代码为sm2_verify.c，编译并运行：
```bash
gcc sm2_verify.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
运行没有报错，说明验证签名成功。

### SM2加解密算法编程入门(补充)

SM2公钥加密：
```c
#include <openssl/evp.h>
#include <openssl/err.h>
#include <openssl/pem.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm2_enc(EVP_PKEY *pkey, const void *in, size_t inlen,
                   unsigned char **out, size_t *outlen)
{
    int ok = 0;
    size_t len;
    unsigned char *buf = NULL;
    EVP_PKEY_CTX *pctx = NULL;

    pctx = EVP_PKEY_CTX_new(pkey, NULL);
    if (pctx == NULL)
        return 0;

    if (EVP_PKEY_encrypt_init(pctx) <= 0)
        goto end;

    if (EVP_PKEY_encrypt(pctx, NULL, &len, in, inlen) <= 0)
        goto end;

    buf = OPENSSL_malloc(len);
    if (buf == NULL)
        goto end;

    if (EVP_PKEY_encrypt(pctx, buf, &len, in, inlen) <= 0)
        goto end;

    OPENSSL_free(*out);
    *out = buf;
    buf = NULL;
    *outlen = len;

    ok = 1;
end:
    EVP_PKEY_CTX_free(pctx);
    OPENSSL_free(buf);
    return ok;
}

int main()
{
    const char *msg = "hello tongsuo";
    unsigned char *buf = NULL;
    char *s = NULL;
    size_t len = 0;
    EVP_PKEY *pkey = NULL;
    BIO *bio = NULL;
    unsigned char pubkey[] =
        "-----BEGIN PUBLIC KEY-----\n"
"MFkwEwYHKoZIzj0CAQYIKoEcz1UBgi0DQgAEMKnjZFqe34rtSmZ7g5ALnKTPKYhM\n"
"xEy9cpq3Kzgb7/JoTTZHm9tGrG1oBUCNszq0jPff7Fxp/azNv7rDPzJXGg==\n"
"-----END PUBLIC KEY-----\n";
    int ret;

    bio = BIO_new_mem_buf(pubkey, -1);
    assert(bio != NULL);

    pkey = PEM_read_bio_PUBKEY(bio, NULL, NULL, NULL);
    assert(pkey != NULL);

    ret = sm2_enc(pkey, msg, strlen((const char *)msg), &buf, &len);
    assert(ret == 1);

    s = OPENSSL_buf2hexstr(buf, len);
    printf("SM2_enc(%s)=%s\n", msg, s);

    OPENSSL_free(s);
    OPENSSL_free(buf);
    EVP_PKEY_free(pkey);
    BIO_free(bio);
    return 0;
}
```
保存代码为sm2_enc.c，编译并运行：
```bash
gcc sm2_enc.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
注意：因为SM2加密过程有随机数的参与，所以每次加密的结果并不相同。

SM2私钥解密：
```c
#include <openssl/evp.h>
#include <openssl/err.h>
#include <openssl/pem.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>
#include <assert.h>

static int sm2_dec(EVP_PKEY *pkey, const void *in, size_t inlen,
                   unsigned char **out, size_t *outlen)
{
    int ok = 0;
    size_t len;
    unsigned char *buf = NULL;
    EVP_PKEY_CTX *pctx = NULL;

    pctx = EVP_PKEY_CTX_new(pkey, NULL);
    if (pctx == NULL)
        return 0;

    if (EVP_PKEY_decrypt_init(pctx) <= 0)
        goto end;

    if (EVP_PKEY_decrypt(pctx, NULL, &len, in, inlen) <= 0)
        goto end;

    buf = OPENSSL_malloc(len);
    if (buf == NULL)
        goto end;

    if (EVP_PKEY_decrypt(pctx, buf, &len, in, inlen) <= 0)
        goto end;

    OPENSSL_free(*out);
    *out = buf;
    buf = NULL;
    *outlen = len;

    ok = 1;
end:
    EVP_PKEY_CTX_free(pctx);
    OPENSSL_free(buf);
    return ok;
}

int main()
{
    unsigned char ciphertext[] = {
0x30, 0x76, 0x02, 0x21, 0x00, 0x85, 0x94, 0xA2, 0x50, 0x1D, 0x75, 0x9B, 0x8C,
0xF2, 0x29, 0x48, 0x86, 0xC5, 0xB6, 0x22, 0xAF, 0xDF, 0xAC, 0x82, 0xA0, 0x0C,
0x92, 0x71, 0x7E, 0x69, 0x11, 0x0D, 0x33, 0xD5, 0x9B, 0x3C, 0xEB, 0x02, 0x20,
0x18, 0xCE, 0x4B, 0x1F, 0xF9, 0x11, 0xD4, 0xCC, 0x1E, 0xF7, 0x7D, 0xFF, 0xC1,
0xAE, 0x3C, 0xBA, 0x00, 0x20, 0x9C, 0xE8, 0x8A, 0x4A, 0x01, 0x8A, 0x6C, 0x68,
0xF3, 0x99, 0xD4, 0x3D, 0x75, 0x64, 0x04, 0x20, 0x3B, 0xDE, 0xE9, 0xB4, 0xA0,
0x3D, 0x90, 0xC9, 0xA6, 0x9A, 0xA6, 0x3D, 0x88, 0xB3, 0xF2, 0x3C, 0xFB, 0x53,
0x61, 0x4C, 0x7F, 0x7B, 0xC6, 0x70, 0x4E, 0x06, 0x5F, 0xDB, 0x9A, 0x1C, 0x4D,
0x04, 0x04, 0x0D, 0xCF, 0x7B, 0x21, 0x0C, 0x4B, 0x20, 0xA1, 0x31, 0xE4, 0x79,
0xA6, 0xCF, 0xDC,
};
    unsigned char *buf = NULL;
    size_t i, len = 0;
    EVP_PKEY *pkey = NULL;
    BIO *bio = NULL;
    unsigned char privkey[] =
        "-----BEGIN PRIVATE KEY-----\n"
"MIGHAgEAMBMGByqGSM49AgEGCCqBHM9VAYItBG0wawIBAQQgSKhk+4xGyDI+IS2H\n"
"WVfFPDxh1qv5+wtrddaIsGNXGZihRANCAAQwqeNkWp7fiu1KZnuDkAucpM8piEzE\n"
"TL1ymrcrOBvv8mhNNkeb20asbWgFQI2zOrSM99/sXGn9rM2/usM/Mlca\n"
"-----END PRIVATE KEY-----\n";
    int ret;

    bio = BIO_new_mem_buf(privkey, -1);
    assert(bio != NULL);

    pkey = PEM_read_bio_PrivateKey(bio, NULL, NULL, NULL);
    assert(pkey != NULL);

    ret = sm2_dec(pkey, ciphertext, sizeof(ciphertext), &buf, &len);
    assert(ret == 1);

    printf("SM2_dec()=");

    for (i = 0; i < len; i++)
        printf("%c", buf[i]);

    printf("\n");

    OPENSSL_free(buf);
    EVP_PKEY_free(pkey);
    BIO_free(bio);
    return 0;
}
```
保存代码为sm2_dec.c，编译并运行：
```bash
gcc sm2_dec.c -I/opt/tongsuo/include -L/opt/tongsuo/lib64 -lcrypto -Wl,-rpath=/opt/tongsuo/lib64

./a.out
```
解密出来的明文应该是hello tongsuo，
![image.png](img/handbook8.png)

## 自签发国密证书

签发步骤为：

1. 签发CA根证书
2. 签发中间CA证书
3. 签发服务器国密双证书

### 签发CA证书

**准备：**
```bash
mkdir -p certs/ca
mkdir certs/ca/{newcerts,db,private,crl}
touch certs/ca/crl/crlnumber
echo 00 > certs/ca/crl/crlnumber
touch certs/ca/db/{index,serial}
echo 00 > certs/ca/db/serial
```
ca.cnf文件：
```bash
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./certs/ca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/ca.key
certificate       = $dir/ca.crt

# For certificate revocation lists.
crlnumber         = $dir/crl/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca

req_extensions = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

# Optionally, specify some defaults.
countryName_default             =
stateOrProvinceName_default     =
localityName_default            =
0.organizationName_default      =
#organizationalUnitName_default =
#emailAddress_default           =

[ v3_req ]

# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = test.com

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

[ crl_ext ]
# CRL extensions.
# Only issuerAltName and authorityKeyIdentifier make any sense in a CRL.

# issuerAltName=issuer:copy
authorityKeyIdentifier=keyid:always

```

**签发CA根证书**
```bash
# 生成SM2私钥
/opt/tongsuo/bin/tongsuo genpkey -algorithm ec -out certs/ca/sm2.key -pkeyopt ec_paramgen_curve:sm2

# 生成CSR
/opt/tongsuo/bin/tongsuo req -batch -config ca.cnf -key certs/ca/sm2.key -new -nodes -out certs/ca/sm2.csr -sm3 -subj "/C=AB/ST=CD/L=EF/O=GH/OU=IJ/CN=CA SM2"

# 自签发CA证书
/opt/tongsuo/bin/tongsuo ca -batch -config ca.cnf -days 365 -extensions v3_ca -in certs/ca/sm2.csr -keyfile certs/ca/sm2.key -md sm3 -notext -out certs/ca/sm2.crt -selfsign
```
### 签发中间CA证书

**准备**
```bash
mkdir -p certs/subca
mkdir certs/subca/{newcerts,db,private,crl}
touch certs/subca/crl/crlnumber
echo 00 > certs/subca/crl/crlnumber
touch certs/subca/db/{index,serial}
echo 00 > certs/subca/db/serial
```
subca.cnf文件：
```bash
[ ca ]
# `man ca`
default_ca = CA_default

[ CA_default ]
# Directory and file locations.
dir               = ./certs/subca
certs             = $dir/certs
crl_dir           = $dir/crl
new_certs_dir     = $dir/newcerts
database          = $dir/db/index
unique_subject    = no
serial            = $dir/db/serial
RANDFILE          = $dir/private/random

# The root key and root certificate.
private_key       = $dir/ca.key
certificate       = $dir/ca.crt

# For certificate revocation lists.
crlnumber         = $dir/crl/crlnumber
crl               = $dir/crl/ca.crl.pem
crl_extensions    = crl_ext
default_crl_days  = 30

# SHA-1 is deprecated, so use SHA-2 instead.
default_md        = sha256

name_opt          = ca_default
cert_opt          = ca_default
default_days      = 365
preserve          = no
policy            = policy_strict

[ policy_strict ]
# The root CA should only sign intermediate certificates that match.
# See the POLICY FORMAT section of `man ca`.
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
# Options for the `req` tool (`man req`).
default_bits        = 2048
distinguished_name  = req_distinguished_name
string_mask         = utf8only

# SHA-1 is deprecated, so use SHA-2 instead.
default_md          = sha256

# Extension to add when the -x509 option is used.
x509_extensions     = v3_ca
req_extensions      = v3_req

[ req_distinguished_name ]
# See <https://en.wikipedia.org/wiki/Certificate_signing_request>.
countryName                     = optional
stateOrProvinceName             = optional
localityName                    = optional
0.organizationName              = optional
organizationalUnitName          = optional
commonName                      = optional
emailAddress                    = optional

# Optionally, specify some defaults.
countryName_default             =
stateOrProvinceName_default     =
localityName_default            =
0.organizationName_default      =
#organizationalUnitName_default =
#emailAddress_default           =

[ v3_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = localhost
DNS.2 = localhost.localdomain
DNS.3 = 127.0.0.1

[ v3_ca ]
# Extensions for a typical CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ v3_intermediate_ca ]
# Extensions for a typical intermediate CA (`man x509v3_config`).
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical, CA:true, pathlen:0
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ client_cert ]
# Extensions for client certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = client, email
nsComment = "OpenSSL Generated Client Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = clientAuth, emailProtection

[ server_cert ]
# Extensions for server certificates (`man x509v3_config`).
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Tongsuo Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ server_sign_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Tongsuo Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = nonRepudiation, digitalSignature
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[ server_enc_req ]
# Extensions to add to a certificate request
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "Tongsuo Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = keyAgreement, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[ crl_ext ]
# Extension for CRLs (`man x509v3_config`).
authorityKeyIdentifier=keyid:always

[ ocsp ]
# Extension for OCSP signing certificates (`man ocsp`).
basicConstraints = CA:FALSE
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
keyUsage = critical, digitalSignature
extendedKeyUsage = critical, OCSPSigning

```
**签发中间CA证书**
```bash
# 生成SM2私钥
/opt/tongsuo/bin/tongsuo genpkey -algorithm "ec" -out certs/subca/sm2.key -pkeyopt ec_paramgen_curve:sm2

# 生成CSR
/opt/tongsuo/bin/tongsuo req -batch -config subca.cnf -key certs/subca/sm2.key -new -nodes -out certs/subca/sm2.csr -sm3 -subj "/C=AB/ST=CD/L=EF/O=GH/OU=IJ/CN=SUBCA SM2"

# 使用CA证书签发中间CA证书
/opt/tongsuo/bin/tongsuo ca -batch -cert certs/ca/sm2.crt -config subca.cnf -days 365 -extensions "v3_intermediate_ca" -in certs/subca/sm2.csr -keyfile certs/ca/sm2.key -md sm3 -notext -out certs/subca/sm2.crt

```
### 签发服务器双证书

**准备：**
```bash
mkdir certs/server
```
**签发SM2签名证书：**
```bash
# 生成SM2签名私钥
/opt/tongsuo/bin/tongsuo genpkey -algorithm ec -out certs/server/sm2_sign.key -pkeyopt "ec_paramgen_curve:sm2"

# 生成CSR
/opt/tongsuo/bin/tongsuo req -batch -config subca.cnf -key certs/server/sm2_sign.key -new -nodes -out certs/server/sm2_sign.csr -sm3 -subj "/C=AB/ST=CD/L=EF/O=GH/OU=IJ/CN=SERVER Sign SM2"

# 使用中间CA证书签发签名证书
/opt/tongsuo/bin/tongsuo ca -batch -cert certs/subca/sm2.crt -config subca.cnf -days 365 -extensions server_sign_req -in certs/server/sm2_sign.csr -keyfile certs/subca/sm2.key -md sm3 -notext -out certs/server/sm2_sign.crt
```
**签发SM2加密证书：**
```bash
# 生成SM2加密私钥
/opt/tongsuo/bin/tongsuo genpkey -algorithm ec -out certs/server/sm2_enc.key -pkeyopt "ec_paramgen_curve:sm2"

# 生成CSR
/opt/tongsuo/bin/tongsuo req -batch -config subca.cnf -key certs/server/sm2_enc.key -new -nodes -out certs/server/sm2_enc.csr -sm3 -subj "/C=AB/ST=CD/L=EF/O=GH/OU=IJ/CN=SERVER Enc SM2"

# 使用中间CA证书签发加密证书
/opt/tongsuo/bin/tongsuo ca -batch -cert certs/subca/sm2.crt -config subca.cnf -days 365 -extensions "server_enc_req" -in certs/server/sm2_enc.csr -keyfile certs/subca/sm2.key -md sm3 -notext -out certs/server/sm2_enc.crt
```
### 查看私钥和证书

命令行查看私钥：
```bash
/opt/tongsuo/bin/tongsuo pkey -in certs/server/sm2_sign.key -text -noout
```
结果如下：
![image.png](img/handbook9.png)

命令行查看证书：
```bash
/opt/tongsuo/bin/tongsuo x509 -in certs/server/sm2_sign.crt -text -noout
```
结果如下：
![image.png](img/handbook10.png)

浏览器查看证书，使用支持360浏览器访问[https://ebssec.boc.cn/boc15/login.html](https://ebssec.boc.cn/boc15/login.html)，或者其他支持国密协议的浏览器也可以，查看国密证书，截图如下：

![image.png](img/handbook11.png)

点击证书信息，可以查看详细的信息：

![image.png](img/handbook12.png)

## 实战国密传输协议

铜锁支持使用 `s_client` 命令行发起国密传输协议的握手。可以对照国密TLCP标准熟悉国密协议，详见[http://c.gb688.cn/bzgk/gb/showGb?type=online&hcno=778097598DA2761E94A5FF3F77BD66DA](http://c.gb688.cn/bzgk/gb/showGb?type=online&hcno=778097598DA2761E94A5FF3F77BD66DA)。

通过命令行跟踪国密握手消息，如下所示：
```bash
root@5bd3e64034a7:~# /opt/tongsuo/bin/tongsuo s_client -connect ebssec.boc.cn:443 -enable_ntls -ntls -trace
CONNECTED(00000003)
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 89
    ClientHello, Length=85
      client_version=0x101 (NTLS)
      Random:
        gmt_unix_time=0x459BE7C1
        random_bytes (len=28): F5F7847E20A53C25DF6DC5FACB153593767A26BDDD322D02869EEF5C
      session_id (len=0):
      cipher_suites (len=18)
        {0xE0, 0x53} ECC_SM4_GCM_SM3
        {0xE0, 0x51} ECDHE_SM4_GCM_SM3
        {0xE0, 0x5A} RSA_SM4_GCM_SHA256
        {0xE0, 0x59} RSA_SM4_GCM_SM3
        {0xE0, 0x13} ECC_SM4_CBC_SM3
        {0xE0, 0x11} ECDHE_SM4_CBC_SM3
        {0xE0, 0x1C} RSA_SM4_CBC_SHA256
        {0xE0, 0x19} RSA_SM4_CBC_SM3
        {0x00, 0xFF} TLS_EMPTY_RENEGOTIATION_INFO_SCSV
      compression_methods (len=1)
        No Compression (0x00)
      extensions, length = 26
        extension_type=server_name(0), length=18
          0000 - 00 10 00 00 0d 65 62 73-73 65 63 2e 62 6f 63   .....ebssec.boc
          000f - 2e 63 6e                                       .cn
        extension_type=session_ticket(35), length=0

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 53
    ServerHello, Length=49
      server_version=0x101 (NTLS)
      Random:
        gmt_unix_time=0x64267A0A
        random_bytes (len=28): D5288BA91B99B2ED1115E30A8B8DBA1F0212B609DFBFFBFE20062AA4
      session_id (len=0):
      cipher_suite {0xE0, 0x13} ECC_SM4_CBC_SM3
      compression_method: No Compression (0x00)
      extensions, length = 9
        extension_type=renegotiate(65281), length=1
            <EMPTY>
        extension_type=session_ticket(35), length=0

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 1458
    Certificate, Length=1454
      certificate_list, length=1451
        ASN.1Cert, length=723
------details-----
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 82514097008 (0x1336393370)
        Signature Algorithm: SM2-with-SM3
        Issuer: C = CN, O = CFCA SM2 OCA1
        Validity
            Not Before: Jun 11 09:05:20 2021 GMT
            Not After : Jun 19 08:16:56 2026 GMT
        Subject: C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:fb:0d:52:7a:19:40:cf:42:4a:7b:c2:e7:b4:db:
                    bd:d7:f2:39:30:ae:3c:e4:a5:66:63:c0:cb:10:4a:
                    16:3f:98:d5:01:ff:c6:5b:9b:1d:d5:5f:e5:7a:87:
                    ac:ed:63:08:34:62:ed:a3:79:20:a1:97:40:5d:78:
                    f7:67:3c:d3:73
                ASN1 OID: SM2
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                5C:93:58:20:5A:24:73:56:10:1B:64:50:10:EC:E9:A7:CA:07:41:11
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Certificate Policies:
                Policy: 2.16.156.112554.1.1
                  CPS: http://www.cfca.com.cn/us/us-14.htm
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:http://crl.cfca.com.cn/SM2/crl5618.crl
            X509v3 Subject Alternative Name:
                DNS:ebssec.boc.cn
            X509v3 Key Usage: critical
                Digital Signature, Non Repudiation
            X509v3 Subject Key Identifier:
                9E:A8:16:8F:CE:AC:A8:03:84:71:4E:46:96:AA:D3:89:17:ED:3D:4A
            X509v3 Extended Key Usage:
                TLS Web Client Authentication, TLS Web Server Authentication
    Signature Algorithm: SM2-with-SM3
    Signature Value:
        30:46:02:21:00:af:85:2b:db:bf:98:7a:11:19:75:61:c0:8b:
        83:e7:f3:f5:49:5e:41:b6:8f:7c:16:30:52:35:03:d9:d0:07:
        55:02:21:00:c4:42:e2:4f:52:fe:64:82:d1:4a:54:bc:2a:a1:
        fc:34:02:d9:48:bc:4d:c7:1d:e4:6d:88:81:84:ac:72:75:0d
-----BEGIN CERTIFICATE-----
MIICzzCCAnKgAwIBAgIFEzY5M3AwDAYIKoEcz1UBg3UFADAlMQswCQYDVQQGEwJD
TjEWMBQGA1UECgwNQ0ZDQSBTTTIgT0NBMTAeFw0yMTA2MTEwOTA1MjBaFw0yNjA2
MTkwODE2NTZaMIGRMQswCQYDVQQGEwJDTjEPMA0GA1UECAwG5YyX5LqsMQ8wDQYD
VQQHDAbljJfkuqwxJzAlBgNVBAoMHuS4reWbvemTtuihjOiCoeS7veaciemZkOWF
rOWPuDERMA8GA1UECwwITG9jYWwgUkExDDAKBgNVBAsMA1NTTDEWMBQGA1UEAwwN
ZWJzc2VjLmJvYy5jbjBZMBMGByqGSM49AgEGCCqBHM9VAYItA0IABPsNUnoZQM9C
SnvC57TbvdfyOTCuPOSlZmPAyxBKFj+Y1QH/xlubHdVf5XqHrO1jCDRi7aN5IKGX
QF1492c803OjggEeMIIBGjAfBgNVHSMEGDAWgBRck1ggWiRzVhAbZFAQ7OmnygdB
ETAMBgNVHRMBAf8EAjAAMEgGA1UdIARBMD8wPQYIYIEchu8qAQEwMTAvBggrBgEF
BQcCARYjaHR0cDovL3d3dy5jZmNhLmNvbS5jbi91cy91cy0xNC5odG0wNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5jZmNhLmNvbS5jbi9TTTIvY3JsNTYxOC5j
cmwwGAYDVR0RBBEwD4INZWJzc2VjLmJvYy5jbjAOBgNVHQ8BAf8EBAMCBsAwHQYD
VR0OBBYEFJ6oFo/OrKgDhHFORpaq04kX7T1KMB0GA1UdJQQWMBQGCCsGAQUFBwMC
BggrBgEFBQcDATAMBggqgRzPVQGDdQUAA0kAMEYCIQCvhSvbv5h6ERl1YcCLg+fz
9UleQbaPfBYwUjUD2dAHVQIhAMRC4k9S/mSC0UpUvCqh/DQC2Ui8Tccd5G2IgYSs
cnUN
-----END CERTIFICATE-----
------------------
        ASN.1Cert, length=722
------details-----
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 82514097009 (0x1336393371)
        Signature Algorithm: SM2-with-SM3
        Issuer: C = CN, O = CFCA SM2 OCA1
        Validity
            Not Before: Jun 11 09:05:20 2021 GMT
            Not After : Jun 19 08:16:56 2026 GMT
        Subject: C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:c9:f5:ab:e8:5b:57:48:b5:aa:72:80:cb:b4:1e:
                    67:76:5f:00:3f:a0:a8:75:f8:17:93:2a:22:1b:1a:
                    ac:e0:e5:5a:c6:af:7f:f7:5c:a6:b0:b4:17:6e:fb:
                    cd:ce:38:69:80:41:ff:7b:9c:cb:83:c5:a9:76:91:
                    1d:0a:7c:3c:4c
                ASN1 OID: SM2
        X509v3 extensions:
            X509v3 Authority Key Identifier:
                5C:93:58:20:5A:24:73:56:10:1B:64:50:10:EC:E9:A7:CA:07:41:11
            X509v3 Basic Constraints: critical
                CA:FALSE
            X509v3 Certificate Policies:
                Policy: 2.16.156.112554.1.1
                  CPS: http://www.cfca.com.cn/us/us-14.htm
            X509v3 CRL Distribution Points:
                Full Name:
                  URI:http://crl.cfca.com.cn/SM2/crl5618.crl
            X509v3 Subject Alternative Name:
                DNS:ebssec.boc.cn
            X509v3 Key Usage: critical
                Key Encipherment, Data Encipherment, Key Agreement
            X509v3 Subject Key Identifier:
                5F:DA:D4:91:EF:CC:BC:DB:A4:56:C1:96:35:FB:84:DC:51:A6:3F:F6
            X509v3 Extended Key Usage:
                TLS Web Client Authentication, TLS Web Server Authentication
    Signature Algorithm: SM2-with-SM3
    Signature Value:
        30:45:02:21:00:c2:38:58:b5:79:97:20:88:de:ad:fa:1e:a5:
        c4:bc:12:82:b0:21:dc:96:a5:97:e6:72:03:67:8f:c3:ac:5c:
        8f:02:20:37:20:ef:a3:be:b5:76:9c:09:85:cc:96:7f:25:42:
        02:76:93:7f:45:5f:e0:32:d6:23:52:be:4b:ba:68:52:bf
-----BEGIN CERTIFICATE-----
MIICzjCCAnKgAwIBAgIFEzY5M3EwDAYIKoEcz1UBg3UFADAlMQswCQYDVQQGEwJD
TjEWMBQGA1UECgwNQ0ZDQSBTTTIgT0NBMTAeFw0yMTA2MTEwOTA1MjBaFw0yNjA2
MTkwODE2NTZaMIGRMQswCQYDVQQGEwJDTjEPMA0GA1UECAwG5YyX5LqsMQ8wDQYD
VQQHDAbljJfkuqwxJzAlBgNVBAoMHuS4reWbvemTtuihjOiCoeS7veaciemZkOWF
rOWPuDERMA8GA1UECwwITG9jYWwgUkExDDAKBgNVBAsMA1NTTDEWMBQGA1UEAwwN
ZWJzc2VjLmJvYy5jbjBZMBMGByqGSM49AgEGCCqBHM9VAYItA0IABMn1q+hbV0i1
qnKAy7QeZ3ZfAD+gqHX4F5MqIhsarODlWsavf/dcprC0F277zc44aYBB/3ucy4PF
qXaRHQp8PEyjggEeMIIBGjAfBgNVHSMEGDAWgBRck1ggWiRzVhAbZFAQ7OmnygdB
ETAMBgNVHRMBAf8EAjAAMEgGA1UdIARBMD8wPQYIYIEchu8qAQEwMTAvBggrBgEF
BQcCARYjaHR0cDovL3d3dy5jZmNhLmNvbS5jbi91cy91cy0xNC5odG0wNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5jZmNhLmNvbS5jbi9TTTIvY3JsNTYxOC5j
cmwwGAYDVR0RBBEwD4INZWJzc2VjLmJvYy5jbjAOBgNVHQ8BAf8EBAMCAzgwHQYD
VR0OBBYEFF/a1JHvzLzbpFbBljX7hNxRpj/2MB0GA1UdJQQWMBQGCCsGAQUFBwMC
BggrBgEFBQcDATAMBggqgRzPVQGDdQUAA0gAMEUCIQDCOFi1eZcgiN6t+h6lxLwS
grAh3Jall+ZyA2ePw6xcjwIgNyDvo761dpwJhcyWfyVCAnaTf0Vf4DLWI1K+S7po
Ur8=
-----END CERTIFICATE-----
------------------

depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
verify return:1
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 77
    ServerKeyExchange, Length=73
      KeyExchangeAlgorithm=SM2
      Signature (len=71): 304502210081ED06241B7AA30E46B6E2FEA3DA69C3AF8032535CB7314AFC99A08A0301380102202A938E95A7BC224205B7E9740171CB064D02299B9A5BECBB03DAADCF963A09E8

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 4
    ServerHelloDone, Length=0

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 162
    ClientKeyExchange, Length=158
      KeyExchangeAlgorithm=SM2
        EncryptedPreMasterSecret (len=156): 308199022100D777F6AE60EF701B7CBFF4D1AA5D3611319FD4619059030F4A262ECBB9371F4502207EAAE6B5308403B40899EF6C7D7C29CA6213E00992E2DF8C0D2025B5B2F453A9042052975DA78D3E9DDB09F6B4CC0D1B7F215DD83EDA250E399EDC01BFDE6F4B864D0430870689C67CFA4FDFA11E1615E9D15054E0E000B3552F1D1D9B3DA7752CB87379143235AF4C692888D16D23DE15EC62C4

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ChangeCipherSpec (20)
  Length = 1
    change_cipher_spec (1)

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 80
    Finished, Length=12
      verify_data (len=12): DFA08D70D3ADCFD49B7E327C

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 202
    NewSessionTicket, Length=198
        ticket_lifetime_hint=300
        ticket (len=192): E4D1199438A33B9FA1B84D9A01B154F8539D43208BD71A1B861DD5605487FEAE862FE165A1D5F03ED325B7E53086B1C3DC52CF730D533B20F07EE8D7582A3F88FADCA2E144B515A2BC8C4B85E94E5D75FE0ECB7C628EC85BFD29A6AD473E8737AACA08A8F6E7EF6178B3984DE1B7B5A0B707598D98B57E948876A8E48E8979BE2F8DB285A4137F6E38267598A5C3BB47881A47C971C16B7C9BAFA28E3AD65B53D68BB44CD35560BDD170D6100ACC7E5B6002AAA5B1B69CB48F59B0236953068E

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 80
    Finished, Length=12
      verify_data (len=12): D6E49646D1BF6C4C6B587373

---
Certificate chain
 0 s:C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
   i:C = CN, O = CFCA SM2 OCA1
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: SM2-SM3
   v:NotBefore: Jun 11 09:05:20 2021 GMT; NotAfter: Jun 19 08:16:56 2026 GMT
 1 s:C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
   i:C = CN, O = CFCA SM2 OCA1
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: SM2-SM3
   v:NotBefore: Jun 11 09:05:20 2021 GMT; NotAfter: Jun 19 08:16:56 2026 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIICzzCCAnKgAwIBAgIFEzY5M3AwDAYIKoEcz1UBg3UFADAlMQswCQYDVQQGEwJD
TjEWMBQGA1UECgwNQ0ZDQSBTTTIgT0NBMTAeFw0yMTA2MTEwOTA1MjBaFw0yNjA2
MTkwODE2NTZaMIGRMQswCQYDVQQGEwJDTjEPMA0GA1UECAwG5YyX5LqsMQ8wDQYD
VQQHDAbljJfkuqwxJzAlBgNVBAoMHuS4reWbvemTtuihjOiCoeS7veaciemZkOWF
rOWPuDERMA8GA1UECwwITG9jYWwgUkExDDAKBgNVBAsMA1NTTDEWMBQGA1UEAwwN
ZWJzc2VjLmJvYy5jbjBZMBMGByqGSM49AgEGCCqBHM9VAYItA0IABPsNUnoZQM9C
SnvC57TbvdfyOTCuPOSlZmPAyxBKFj+Y1QH/xlubHdVf5XqHrO1jCDRi7aN5IKGX
QF1492c803OjggEeMIIBGjAfBgNVHSMEGDAWgBRck1ggWiRzVhAbZFAQ7OmnygdB
ETAMBgNVHRMBAf8EAjAAMEgGA1UdIARBMD8wPQYIYIEchu8qAQEwMTAvBggrBgEF
BQcCARYjaHR0cDovL3d3dy5jZmNhLmNvbS5jbi91cy91cy0xNC5odG0wNwYDVR0f
BDAwLjAsoCqgKIYmaHR0cDovL2NybC5jZmNhLmNvbS5jbi9TTTIvY3JsNTYxOC5j
cmwwGAYDVR0RBBEwD4INZWJzc2VjLmJvYy5jbjAOBgNVHQ8BAf8EBAMCBsAwHQYD
VR0OBBYEFJ6oFo/OrKgDhHFORpaq04kX7T1KMB0GA1UdJQQWMBQGCCsGAQUFBwMC
BggrBgEFBQcDATAMBggqgRzPVQGDdQUAA0kAMEYCIQCvhSvbv5h6ERl1YcCLg+fz
9UleQbaPfBYwUjUD2dAHVQIhAMRC4k9S/mSC0UpUvCqh/DQC2Ui8Tccd5G2IgYSs
cnUN
-----END CERTIFICATE-----
subject=C = CN, ST = \E5\8C\97\E4\BA\AC, L = \E5\8C\97\E4\BA\AC, O = \E4\B8\AD\E5\9B\BD\E9\93\B6\E8\A1\8C\E8\82\A1\E4\BB\BD\E6\9C\89\E9\99\90\E5\85\AC\E5\8F\B8, OU = Local RA, OU = SSL, CN = ebssec.boc.cn
issuer=C = CN, O = CFCA SM2 OCA1
---
No client certificate CA names sent
Peer signing digest: SM3
Peer signature type: SM2
---
SSL handshake has read 1910 bytes and written 352 bytes
Verification error: unable to verify the first certificate
---
New, NTLSv1.1, Cipher is ECC-SM2-SM4-CBC-SM3
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : NTLSv1.1
    Cipher    : ECC-SM2-SM4-CBC-SM3
    Session-ID: 34FC1DE275D8626273C0809DD12A7970BBFF6BD4DA7EA1FD757BE3B53856640B
    Session-ID-ctx:
    Master-Key: 3CD21EC27CFE094C4E1EE848DF909C3BF08A32A373D4C2D508B2CF37858D3120F71AB361A134A6C4DAD3B6CA65F5AC27
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - e4 d1 19 94 38 a3 3b 9f-a1 b8 4d 9a 01 b1 54 f8   ....8.;...M...T.
    0010 - 53 9d 43 20 8b d7 1a 1b-86 1d d5 60 54 87 fe ae   S.C .......`T...
    0020 - 86 2f e1 65 a1 d5 f0 3e-d3 25 b7 e5 30 86 b1 c3   ./.e...>.%..0...
    0030 - dc 52 cf 73 0d 53 3b 20-f0 7e e8 d7 58 2a 3f 88   .R.s.S; .~..X*?.
    0040 - fa dc a2 e1 44 b5 15 a2-bc 8c 4b 85 e9 4e 5d 75   ....D.....K..N]u
    0050 - fe 0e cb 7c 62 8e c8 5b-fd 29 a6 ad 47 3e 87 37   ...|b..[.)..G>.7
    0060 - aa ca 08 a8 f6 e7 ef 61-78 b3 98 4d e1 b7 b5 a0   .......ax..M....
    0070 - b7 07 59 8d 98 b5 7e 94-88 76 a8 e4 8e 89 79 be   ..Y...~..v....y.
    0080 - 2f 8d b2 85 a4 13 7f 6e-38 26 75 98 a5 c3 bb 47   /......n8&u....G
    0090 - 88 1a 47 c9 71 c1 6b 7c-9b af a2 8e 3a d6 5b 53   ..G.q.k|....:.[S
    00a0 - d6 8b b4 4c d3 55 60 bd-d1 70 d6 10 0a cc 7e 5b   ...L.U`..p....~[
    00b0 - 60 02 aa a5 b1 b6 9c b4-8f 59 b0 23 69 53 06 8e   `........Y.#iS..

    Start Time: 1680243210
    Timeout   : 7200 (sec)
    Verify return code: 21 (unable to verify the first certificate)
    Extended master secret: no
    QUIC: no
---
```
也可以通过支持国密协议的浏览器访问国密HTTPS服务器，并通过wireshark抓包软件查看协议消息。

通过360浏览器访问[https://ebssec.boc.cn/](https://ebssec.boc.cn/)，如图所示：

![image.png](img/handbook13.png)

可以从截图中看出使用的国密传输协议。

同时使用 Wireshark 进行抓包，可以看到 TLCP 的握手消息的详细内容。

![image.png](img/handbook14.png)

## 国密传输协议编程入门

### 国密服务端

server.c
```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

int main(int argc, char **argv)
{
    struct sockaddr_in addr;
    unsigned int addr_len = sizeof(addr);
    const SSL_METHOD *method;
    SSL_CTX *ssl_ctx = NULL;
    SSL *ssl = NULL;
    int fd = -1, conn_fd = -1;
    char *txbuf = NULL;
    size_t txcap = 0;
    int txlen;
    char rxbuf[128];
    size_t rxcap = sizeof(rxbuf);
    int rxlen;
    char *server_ip = "127.0.0.1";
    char *server_port = "443";
    int server_running = 1;
    int optval = 1;

    if (argc == 2) {
        server_ip = argv[1];
        server_port = strstr(argv[1], ":");
        if (server_port != NULL)
            *server_port++ = '\0';
        else
            server_port = "443";
    }

    method = NTLS_server_method();
    ssl_ctx = SSL_CTX_new(method);
    if (ssl_ctx == NULL) {
        perror("Unable to create SSL context");
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    SSL_CTX_enable_ntls(ssl_ctx);

    /* Set the key and cert */
    if (!SSL_CTX_use_sign_certificate_file(ssl_ctx, "certs/server/sm2_sign.crt",
                                           SSL_FILETYPE_PEM)
        || !SSL_CTX_use_sign_PrivateKey_file(ssl_ctx, "certs/server/sm2_sign.key",
                                             SSL_FILETYPE_PEM)
	    || !SSL_CTX_use_enc_certificate_file(ssl_ctx, "certs/server/sm2_enc.crt",
                                              SSL_FILETYPE_PEM)
	    || !SSL_CTX_use_enc_PrivateKey_file(ssl_ctx, "certs/server/sm2_enc.key",
                                             SSL_FILETYPE_PEM)) {
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("Unable to create socket");
        exit(EXIT_FAILURE);
    }

    addr.sin_family = AF_INET;
    inet_pton(AF_INET, server_ip, &addr.sin_addr.s_addr);
    addr.sin_port = htons(atoi(server_port));

    /* Reuse the address; good for quick restarts */
    if (setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval)) < 0) {
        perror("setsockopt(SO_REUSEADDR) failed");
        exit(EXIT_FAILURE);
    }

    if (bind(fd, (struct sockaddr*) &addr, sizeof(addr)) < 0) {
        perror("Unable to bind");
        exit(EXIT_FAILURE);
    }

    if (listen(fd, 1) < 0) {
        perror("Unable to listen");
        exit(EXIT_FAILURE);
    }

    printf("We are the server on port: %d\n\n", atoi(server_port));
    /*
     * Loop to accept clients.
     * Need to implement timeouts on TCP & SSL connect/read functions
     * before we can catch a CTRL-C and kill the server.
     */
    while (server_running) {
        /* Wait for TCP connection from client */
        conn_fd= accept(fd, (struct sockaddr*) &addr, &addr_len);
        if (conn_fd < 0) {
            perror("Unable to accept");
            exit(EXIT_FAILURE);
        }

        printf("Client TCP connection accepted\n");

        /* Create server SSL structure using newly accepted client socket */
        ssl = SSL_new(ssl_ctx);
        SSL_set_fd(ssl, conn_fd);

        /* Wait for SSL connection from the client */
        if (SSL_accept(ssl) <= 0) {
            ERR_print_errors_fp(stderr);
            server_running = 0;
        } else {

            printf("Client TLCP connection accepted\n\n");

            /* Echo loop */
            while (1) {
                /* Get message from client; will fail if client closes connection */
                if ((rxlen = SSL_read(ssl, rxbuf, rxcap)) <= 0) {
                    if (rxlen == 0) {
                        printf("Client closed connection\n");
                    }
                    ERR_print_errors_fp(stderr);
                    break;
                }
                /* Insure null terminated input */
                rxbuf[rxlen] = 0;
                /* Look for kill switch */
                if (strcmp(rxbuf, "kill\n") == 0) {
                    /* Terminate...with extreme prejudice */
                    printf("Server received 'kill' command\n");
                    server_running = 0;
                    break;
                }
                /* Show received message */
                printf("Received: %s", rxbuf);
                /* Echo it back */
                if (SSL_write(ssl, rxbuf, rxlen) <= 0) {
                    ERR_print_errors_fp(stderr);
                }
            }
        }
        if (server_running) {
            /* Cleanup for next client */
            SSL_shutdown(ssl);
            SSL_free(ssl);
            close(conn_fd);
	        conn_fd = -1;
        }
    }
    printf("Server exiting...\n");

exit:
    /* Close up */
    if (ssl != NULL) {
        SSL_shutdown(ssl);
        SSL_free(ssl);
    }
    SSL_CTX_free(ssl_ctx);

    if (conn_fd != -1)
        close(conn_fd);
    if (fd != -1)
        close(fd);

    if (txbuf != NULL && txcap > 0)
        free(txbuf);

    return 0;
}
// gcc server.c  -I/opt/tongsuo/include/ -L/opt/tongsuo/lib64/ -lssl -lcrypto -Wl,-rpath=/opt/tongsuo/lib64 -o server

```
编译服务端，可以使用s_client进行国密握手测试：
```c
gcc server.c  -I/opt/tongsuo/include/ -L/opt/tongsuo/lib64/ -lssl -lcrypto -Wl,-rpath=/opt/tongsuo/lib64 -o server

./server 127.0.0.1:443
```
再启动一个终端，启动客户端：
```bash
/opt/tongsuo/bin/tongsuo s_client -connect 127.0.0.1:443 -enable_ntls -ntls
```
并在客户端发送消息，服务端截图：

![image.png](img/handbook15.png)

### 国密客户端

client.c
```c
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <openssl/ssl.h>
#include <openssl/err.h>

int main(int argc, char **argv)
{
    struct sockaddr_in addr;
    const SSL_METHOD *method;
    SSL_CTX *ssl_ctx = NULL;
    SSL *ssl = NULL;
    int fd = -1;
    char *txbuf = NULL;
    size_t txcap = 0;
    int txlen;
    char rxbuf[128];
    size_t rxcap = sizeof(rxbuf);
    int rxlen;
    char *server_ip = "127.0.0.1";
    char *server_port = "443";

    if (argc == 2) {
        server_ip = argv[1];
        server_port = strstr(argv[1], ":");
        if (server_port != NULL)
            *server_port++ = '\0';
        else
            server_port = "443";
    }

    method = NTLS_client_method();
    ssl_ctx = SSL_CTX_new(method);
    if (ssl_ctx == NULL) {
        perror("Unable to create SSL context");
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }

    SSL_CTX_enable_ntls(ssl_ctx);
    SSL_CTX_set_verify(ssl_ctx, SSL_VERIFY_NONE, NULL);

    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("Unable to create socket");
        exit(EXIT_FAILURE);
    }

    addr.sin_family = AF_INET;
    inet_pton(AF_INET, server_ip, &addr.sin_addr.s_addr);
    addr.sin_port = htons(atoi(server_port));

    if (connect(fd, (struct sockaddr*) &addr, sizeof(addr)) != 0) {
        perror("Unable to TCP connect to server");
        goto exit;
    } else {
        printf("TCP connection to server successful\n");
    }

    /* Create client SSL structure using dedicated client socket */
    ssl = SSL_new(ssl_ctx);
    SSL_set_fd(ssl, fd);

    if (SSL_connect(ssl) == 1) {
        printf("TLCP connection to server successful\n\n");

        /* Loop to send input from keyboard */
        while (1) {
            /* Get a line of input */
            txlen = getline(&txbuf, &txcap, stdin);
            /* Exit loop on error */
            if (txlen < 0 || txbuf == NULL) {
                break;
            }
            /* Exit loop if just a carriage return */
            if (txbuf[0] == '\n') {
                break;
            }
            /* Send it to the server */
            if (SSL_write(ssl, txbuf, txlen) <= 0) {
                printf("Server closed connection\n");
                ERR_print_errors_fp(stderr);
                break;
            }

            /* Wait for the echo */
            rxlen = SSL_read(ssl, rxbuf, rxcap);
            if (rxlen <= 0) {
                printf("Server closed connection\n");
                ERR_print_errors_fp(stderr);
                break;
            } else {
                /* Show it */
                rxbuf[rxlen] = 0;
                printf("Received: %s", rxbuf);
            }
        }
        printf("Client exiting...\n");
    } else {

        printf("SSL connection to server failed\n\n");

        ERR_print_errors_fp(stderr);
    }
exit:
    if (ssl != NULL) {
        SSL_shutdown(ssl);
        SSL_free(ssl);
    }
    SSL_CTX_free(ssl_ctx);

    if (fd != -1)
        close(fd);
	if (txbuf != NULL && txcap > 0)
        free(txbuf);

    return 0;
}
// gcc client.c  -I/opt/tongsuo/include/ -L/opt/tongsuo/lib64/ -lssl -lcrypto -Wl,-rpath=/opt/tongsuo/lib64 -o client
```
编译客户端，使用client连接国密服务器：
```bash
gcc client.c  -I/opt/tongsuo/include/ -L/opt/tongsuo/lib64/ -lssl -lcrypto -Wl,-rpath=/opt/tongsuo/lib64 -o client

./client 127.0.0.1:443
```
可以看到连接成功，发送消息，并收到应答：

![image.png](img/handbook16.png)

## Tengine + 铜锁，搭建国密服务器

下载 Tengine 代码，基于铜锁源码构建 Tengine，如下：
```bash
# Tengine依赖pcre和zlib
apt install libpcre3 libpcre3-dev zlib1g zlib1g-dev

git clone https://github.com/alibaba/tengine

cd tengine

./configure --add-module=modules/ngx_tongsuo_ntls \
    --with-openssl=../Tongsuo \
    --with-openssl-opt="--strict-warnings --api=1.1.1 enable-ntls --debug" \
    --with-http_ssl_module --with-stream \
    --with-stream_ssl_module --with-stream_sni --with-debug

make -j

make install
```
配置Tengine，支持国密HTTPS。配置文件/usr/local/nginx/conf/nginx.conf内容如下：
```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    server {
        listen       443 ssl;
        server_name  localhost;

        enable_ntls  on;
        ssl_sign_certificate        sm2_sign.crt;
        ssl_sign_certificate_key    sm2_sign.key;
        ssl_enc_certificate         sm2_enc.crt;
        ssl_enc_certificate_key     sm2_enc.key;

        location / {
            return 200 "body $ssl_protocol:$ssl_cipher";
        }
    }
}

stream {
     server {
        listen       8443 ssl;

        enable_ntls  on;
        ssl_sign_certificate        sm2_sign.crt;
        ssl_sign_certificate_key    sm2_sign.key;
        ssl_enc_certificate         sm2_enc.crt;
        ssl_enc_certificate_key     sm2_enc.key;

        return "body $ssl_protocol:$ssl_cipher";
    }
}
```
Tengine服务端需要配置国密双证书，可以将之前已经签发好的国密双证书拷贝到Tengine配置目录中，如下：
```bash
cp certs/server/{sm2_sign.crt,sm2_sign.key,sm2_enc.crt,sm2_enc.key} /usr/local/nginx/conf/
```
启动Tengine：
```bash
/usr/local/nginx/sbin/nginx
```
使用铜锁s_client连接Tengine服务器，进行国密TLCP握手：
```bash
root@5bd3e64034a7:/usr/local/nginx/logs# /opt/tongsuo/bin/tongsuo s_client -connect 127.0.0.1:443 -enable_ntls -ntls -trace
CONNECTED(00000003)
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 67
    ClientHello, Length=63
      client_version=0x101 (NTLS)
      Random:
        gmt_unix_time=0x47FAD1C4
        random_bytes (len=28): B206F971501E4E3E8224E3EA740DDFEEC15D46D6AEFCA8FB24D94729
      session_id (len=0):
      cipher_suites (len=18)
        {0xE0, 0x53} ECC_SM4_GCM_SM3
        {0xE0, 0x51} ECDHE_SM4_GCM_SM3
        {0xE0, 0x5A} RSA_SM4_GCM_SHA256
        {0xE0, 0x59} RSA_SM4_GCM_SM3
        {0xE0, 0x13} ECC_SM4_CBC_SM3
        {0xE0, 0x11} ECDHE_SM4_CBC_SM3
        {0xE0, 0x1C} RSA_SM4_CBC_SHA256
        {0xE0, 0x19} RSA_SM4_CBC_SM3
        {0x00, 0xFF} TLS_EMPTY_RENEGOTIATION_INFO_SCSV
      compression_methods (len=1)
        No Compression (0x00)
      extensions, length = 4
        extension_type=session_ticket(35), length=0

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 48
    ServerHello, Length=44
      server_version=0x101 (NTLS)
      Random:
        gmt_unix_time=0x9190A000
        random_bytes (len=28): EA0C49DACA7461BD31792BEC53B5749842982FD3444F574E47524400
      session_id (len=0):
      cipher_suite {0xE0, 0x53} ECC_SM4_GCM_SM3
      compression_method: No Compression (0x00)
      extensions, length = 4
        extension_type=session_ticket(35), length=0

Can't use SSL_get_servername
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 1419
    Certificate, Length=1415
      certificate_list, length=1412
        ASN.1Cert, length=714
------details-----
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 0 (0x0)
        Signature Algorithm: SM2-with-SM3
        Issuer: C = AB, ST = CD, O = GH, OU = IJ, CN = SUBCA SM2
        Validity
            Not Before: Mar 31 03:23:53 2023 GMT
            Not After : Mar 30 03:23:53 2024 GMT
        Subject: C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:ee:d3:16:d5:c9:22:27:45:3c:a3:88:95:ff:07:
                    d1:b7:ac:c2:b2:c0:1c:40:89:43:95:fc:57:8a:b2:
                    69:d1:1e:07:c9:b2:91:91:00:54:62:cc:d6:72:38:
                    c7:e8:6c:bf:ee:50:7e:4a:60:c9:db:54:14:84:32:
                    36:51:7e:99:f5
                ASN1 OID: SM2
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                Tongsuo Generated Server Certificate
            X509v3 Subject Key Identifier:
                50:FC:DA:5F:F6:09:49:1B:C6:F0:F2:D0:27:58:00:26:3B:6D:6B:04
            X509v3 Authority Key Identifier:
                keyid:66:07:34:59:34:12:54:CB:27:12:E0:67:76:33:19:1D:63:61:D8:D1
                DirName:/C=AB/ST=CD/O=GH/OU=IJ/CN=CA SM2
                serial:01
            X509v3 Key Usage:
                Digital Signature, Non Repudiation
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                DNS:localhost, DNS:localhost.localdomain, DNS:127.0.0.1
    Signature Algorithm: SM2-with-SM3
    Signature Value:
        30:45:02:20:54:8c:51:7a:3a:58:6d:77:fb:53:10:06:a7:8b:
        4a:ba:90:d0:54:81:12:9c:ae:41:8a:ac:2c:a5:d6:a9:ae:67:
        02:21:00:9b:c2:48:c5:bd:65:ef:3c:50:08:1e:99:48:40:7b:
        57:34:0d:4b:ba:05:7e:85:b4:65:30:32:7e:ba:29:6a:2a
-----BEGIN CERTIFICATE-----
MIICxjCCAmygAwIBAgIBADAKBggqgRzPVQGDdTBIMQswCQYDVQQGEwJBQjELMAkG
A1UECAwCQ0QxCzAJBgNVBAoMAkdIMQswCQYDVQQLDAJJSjESMBAGA1UEAwwJU1VC
Q0EgU00yMB4XDTIzMDMzMTAzMjM1M1oXDTI0MDMzMDAzMjM1M1owTjELMAkGA1UE
BhMCQUIxCzAJBgNVBAgMAkNEMQswCQYDVQQKDAJHSDELMAkGA1UECwwCSUoxGDAW
BgNVBAMMD1NFUlZFUiBTaWduIFNNMjBZMBMGByqGSM49AgEGCCqBHM9VAYItA0IA
BO7TFtXJIidFPKOIlf8H0beswrLAHECJQ5X8V4qyadEeB8mykZEAVGLM1nI4x+hs
v+5QfkpgydtUFIQyNlF+mfWjggE/MIIBOzAJBgNVHRMEAjAAMBEGCWCGSAGG+EIB
AQQEAwIGQDAzBglghkgBhvhCAQ0EJhYkVG9uZ3N1byBHZW5lcmF0ZWQgU2VydmVy
IENlcnRpZmljYXRlMB0GA1UdDgQWBBRQ/Npf9glJG8bw8tAnWAAmO21rBDBtBgNV
HSMEZjBkgBRmBzRZNBJUyycS4Gd2MxkdY2HY0aFJpEcwRTELMAkGA1UEBhMCQUIx
CzAJBgNVBAgMAkNEMQswCQYDVQQKDAJHSDELMAkGA1UECwwCSUoxDzANBgNVBAMM
BkNBIFNNMoIBATALBgNVHQ8EBAMCBsAwEwYDVR0lBAwwCgYIKwYBBQUHAwEwNgYD
VR0RBC8wLYIJbG9jYWxob3N0ghVsb2NhbGhvc3QubG9jYWxkb21haW6CCTEyNy4w
LjAuMTAKBggqgRzPVQGDdQNIADBFAiBUjFF6Olhtd/tTEAani0q6kNBUgRKcrkGK
rCyl1qmuZwIhAJvCSMW9Ze88UAgemUhAe1c0DUu6BX6FtGUwMn66KWoq
-----END CERTIFICATE-----
------------------
        ASN.1Cert, length=692
------details-----
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
        Signature Algorithm: SM2-with-SM3
        Issuer: C = AB, ST = CD, O = GH, OU = IJ, CN = SUBCA SM2
        Validity
            Not Before: Mar 31 03:24:06 2023 GMT
            Not After : Mar 30 03:24:06 2024 GMT
        Subject: C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Enc SM2
        Subject Public Key Info:
            Public Key Algorithm: id-ecPublicKey
                Public-Key: (256 bit)
                pub:
                    04:be:a4:dc:d4:ef:c4:65:e0:67:c6:ed:d3:fd:e1:
                    39:03:25:15:a3:66:f4:bd:c6:ec:68:e1:0b:3d:7c:
                    e3:aa:83:b0:ec:80:79:f1:0f:7e:5b:7b:95:62:96:
                    8d:17:fd:31:43:cc:11:0c:db:56:d1:71:48:32:5e:
                    b3:f6:23:96:e9
                ASN1 OID: SM2
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Cert Type:
                SSL Server
            Netscape Comment:
                Tongsuo Generated Server Certificate
            X509v3 Subject Key Identifier:
                4A:59:C2:53:27:C7:CE:25:5D:88:70:BE:52:51:3C:96:E0:96:EC:40
            X509v3 Authority Key Identifier:
                keyid:66:07:34:59:34:12:54:CB:27:12:E0:67:76:33:19:1D:63:61:D8:D1
                DirName:/C=AB/ST=CD/O=GH/OU=IJ/CN=CA SM2
                serial:01
            X509v3 Key Usage:
                Key Encipherment, Data Encipherment, Key Agreement
            X509v3 Subject Alternative Name:
                DNS:localhost, DNS:localhost.localdomain, DNS:127.0.0.1
    Signature Algorithm: SM2-with-SM3
    Signature Value:
        30:45:02:21:00:ca:a7:b0:37:8e:e0:11:d2:33:2a:e4:35:69:
        58:38:d9:b4:28:db:eb:92:9f:c9:e0:24:ff:9d:62:f3:b6:6e:
        07:02:20:06:2e:4f:e2:ba:d1:31:07:c9:df:ab:53:8d:db:42:
        13:2d:4c:20:fa:60:d1:38:b5:1f:85:12:2c:e4:7d:70:95
-----BEGIN CERTIFICATE-----
MIICsDCCAlagAwIBAgIBATAKBggqgRzPVQGDdTBIMQswCQYDVQQGEwJBQjELMAkG
A1UECAwCQ0QxCzAJBgNVBAoMAkdIMQswCQYDVQQLDAJJSjESMBAGA1UEAwwJU1VC
Q0EgU00yMB4XDTIzMDMzMTAzMjQwNloXDTI0MDMzMDAzMjQwNlowTTELMAkGA1UE
BhMCQUIxCzAJBgNVBAgMAkNEMQswCQYDVQQKDAJHSDELMAkGA1UECwwCSUoxFzAV
BgNVBAMMDlNFUlZFUiBFbmMgU00yMFkwEwYHKoZIzj0CAQYIKoEcz1UBgi0DQgAE
vqTc1O/EZeBnxu3T/eE5AyUVo2b0vcbsaOELPXzjqoOw7IB58Q9+W3uVYpaNF/0x
Q8wRDNtW0XFIMl6z9iOW6aOCASowggEmMAkGA1UdEwQCMAAwEQYJYIZIAYb4QgEB
BAQDAgZAMDMGCWCGSAGG+EIBDQQmFiRUb25nc3VvIEdlbmVyYXRlZCBTZXJ2ZXIg
Q2VydGlmaWNhdGUwHQYDVR0OBBYEFEpZwlMnx84lXYhwvlJRPJbgluxAMG0GA1Ud
IwRmMGSAFGYHNFk0ElTLJxLgZ3YzGR1jYdjRoUmkRzBFMQswCQYDVQQGEwJBQjEL
MAkGA1UECAwCQ0QxCzAJBgNVBAoMAkdIMQswCQYDVQQLDAJJSjEPMA0GA1UEAwwG
Q0EgU00yggEBMAsGA1UdDwQEAwIDODA2BgNVHREELzAtgglsb2NhbGhvc3SCFWxv
Y2FsaG9zdC5sb2NhbGRvbWFpboIJMTI3LjAuMC4xMAoGCCqBHM9VAYN1A0gAMEUC
IQDKp7A3juAR0jMq5DVpWDjZtCjb65KfyeAk/51i87ZuBwIgBi5P4rrRMQfJ36tT
jdtCEy1MIPpg0Ti1H4USLOR9cJU=
-----END CERTIFICATE-----
------------------

depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Enc SM2
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Enc SM2
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Enc SM2
verify return:1
depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
verify error:num=20:unable to get local issuer certificate
verify return:1
depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
verify error:num=21:unable to verify the first certificate
verify return:1
depth=0 C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
verify return:1
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 76
    ServerKeyExchange, Length=72
      KeyExchangeAlgorithm=SM2
      Signature (len=70): 304402207DF7D8AD1D76DDA96B086DD4D96A821A8949414C8DEA7DE1E2FFD75CE63EDFB502207FE7D0D33B4C199BDA64DB00E781B3999FD022AABAA314949314DA790FD7B28D

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 4
    ServerHelloDone, Length=0

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 162
    ClientKeyExchange, Length=158
      KeyExchangeAlgorithm=SM2
        EncryptedPreMasterSecret (len=156): 3081990220341F92984B61637A9986F9D93A68AF6DF90F2B5773188DEB8FAC48F66F02741A022100A0E1D285873F8B4515C9C166E91DDB7ACF9923BEE84B699D2C4625C794DC6FB2042005CD199F92A696CDF99A07C502D665FED30406D8BD17B471FB0D4754458A625A04308B19E38881425A53DB924C178B9ABD47971E6C51CD8F6782BD0107F98DF520CC6A1F9E6EFA12802F7AEB256B2CD309A4

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ChangeCipherSpec (20)
  Length = 1
    change_cipher_spec (1)

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
      verify_data (len=12): 056EA78F6DA9FA77ED1082BC

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 186
    NewSessionTicket, Length=182
        ticket_lifetime_hint=300
        ticket (len=176): 163ABB8CACB0E6350A435A79D27003573A03C67417073032424E8A9B8D021BA2DDE448AF6110E2C8075F1EE55E25EEF934E38A01B4F97C0D154E570CB502366E00A9F752C2A2664834E6D0189596082BD633068F955A40186D93434AAD682FFA868A2E360D8B776C02EDB1F948A7CEC5AF77A3FD7B1821B5B713EDEB82E44892D153C40E27027BEFC5485D5C2097B5046B0015373E17C3D6E957401D956F7A1CB5CF7E1F3B7A1A0BA0A6A5D495AFC23B

Received Record
Header:
  Version = NTLS (0x101)
  Content Type = ChangeCipherSpec (20)
  Length = 1
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Handshake (22)
  Length = 40
    Finished, Length=12
      verify_data (len=12): 8DAED384E7F1AA04434A223C

---
Certificate chain
 0 s:C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
   i:C = AB, ST = CD, O = GH, OU = IJ, CN = SUBCA SM2
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: SM2-SM3
   v:NotBefore: Mar 31 03:23:53 2023 GMT; NotAfter: Mar 30 03:23:53 2024 GMT
 1 s:C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Enc SM2
   i:C = AB, ST = CD, O = GH, OU = IJ, CN = SUBCA SM2
   a:PKEY: id-ecPublicKey, 256 (bit); sigalg: SM2-SM3
   v:NotBefore: Mar 31 03:24:06 2023 GMT; NotAfter: Mar 30 03:24:06 2024 GMT
---
Server certificate
-----BEGIN CERTIFICATE-----
MIICxjCCAmygAwIBAgIBADAKBggqgRzPVQGDdTBIMQswCQYDVQQGEwJBQjELMAkG
A1UECAwCQ0QxCzAJBgNVBAoMAkdIMQswCQYDVQQLDAJJSjESMBAGA1UEAwwJU1VC
Q0EgU00yMB4XDTIzMDMzMTAzMjM1M1oXDTI0MDMzMDAzMjM1M1owTjELMAkGA1UE
BhMCQUIxCzAJBgNVBAgMAkNEMQswCQYDVQQKDAJHSDELMAkGA1UECwwCSUoxGDAW
BgNVBAMMD1NFUlZFUiBTaWduIFNNMjBZMBMGByqGSM49AgEGCCqBHM9VAYItA0IA
BO7TFtXJIidFPKOIlf8H0beswrLAHECJQ5X8V4qyadEeB8mykZEAVGLM1nI4x+hs
v+5QfkpgydtUFIQyNlF+mfWjggE/MIIBOzAJBgNVHRMEAjAAMBEGCWCGSAGG+EIB
AQQEAwIGQDAzBglghkgBhvhCAQ0EJhYkVG9uZ3N1byBHZW5lcmF0ZWQgU2VydmVy
IENlcnRpZmljYXRlMB0GA1UdDgQWBBRQ/Npf9glJG8bw8tAnWAAmO21rBDBtBgNV
HSMEZjBkgBRmBzRZNBJUyycS4Gd2MxkdY2HY0aFJpEcwRTELMAkGA1UEBhMCQUIx
CzAJBgNVBAgMAkNEMQswCQYDVQQKDAJHSDELMAkGA1UECwwCSUoxDzANBgNVBAMM
BkNBIFNNMoIBATALBgNVHQ8EBAMCBsAwEwYDVR0lBAwwCgYIKwYBBQUHAwEwNgYD
VR0RBC8wLYIJbG9jYWxob3N0ghVsb2NhbGhvc3QubG9jYWxkb21haW6CCTEyNy4w
LjAuMTAKBggqgRzPVQGDdQNIADBFAiBUjFF6Olhtd/tTEAani0q6kNBUgRKcrkGK
rCyl1qmuZwIhAJvCSMW9Ze88UAgemUhAe1c0DUu6BX6FtGUwMn66KWoq
-----END CERTIFICATE-----
subject=C = AB, ST = CD, O = GH, OU = IJ, CN = SERVER Sign SM2
issuer=C = AB, ST = CD, O = GH, OU = IJ, CN = SUBCA SM2
---
No client certificate CA names sent
Peer signing digest: SM3
Peer signature type: SM2
---
SSL handshake has read 1809 bytes and written 290 bytes
Verification error: unable to verify the first certificate
---
New, NTLSv1.1, Cipher is ECC-SM2-SM4-GCM-SM3
Server public key is 256 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : NTLSv1.1
    Cipher    : ECC-SM2-SM4-GCM-SM3
    Session-ID: 78AF7EC6DCA74A509BF5743635B3BC78E25FB07B288063FA9C1D3E67BBE1916A
    Session-ID-ctx:
    Master-Key: 265B3002C6D57CF46E04BA51F2EF7EF7335D58D4F7407D27BAEDA4F57C78D2BC43B95F6E38A6FBF3E67C5364C9D2E618
    PSK identity: None
    PSK identity hint: None
    SRP username: None
    TLS session ticket lifetime hint: 300 (seconds)
    TLS session ticket:
    0000 - 16 3a bb 8c ac b0 e6 35-0a 43 5a 79 d2 70 03 57   .:.....5.CZy.p.W
    0010 - 3a 03 c6 74 17 07 30 32-42 4e 8a 9b 8d 02 1b a2   :..t..02BN......
    0020 - dd e4 48 af 61 10 e2 c8-07 5f 1e e5 5e 25 ee f9   ..H.a...._..^%..
    0030 - 34 e3 8a 01 b4 f9 7c 0d-15 4e 57 0c b5 02 36 6e   4.....|..NW...6n
    0040 - 00 a9 f7 52 c2 a2 66 48-34 e6 d0 18 95 96 08 2b   ...R..fH4......+
    0050 - d6 33 06 8f 95 5a 40 18-6d 93 43 4a ad 68 2f fa   .3...Z@.m.CJ.h/.
    0060 - 86 8a 2e 36 0d 8b 77 6c-02 ed b1 f9 48 a7 ce c5   ...6..wl....H...
    0070 - af 77 a3 fd 7b 18 21 b5-b7 13 ed eb 82 e4 48 92   .w..{.!.......H.
    0080 - d1 53 c4 0e 27 02 7b ef-c5 48 5d 5c 20 97 b5 04   .S..'.{..H]\ ...
    0090 - 6b 00 15 37 3e 17 c3 d6-e9 57 40 1d 95 6f 7a 1c   k..7>....W@..oz.
    00a0 - b5 cf 7e 1f 3b 7a 1a 0b-a0 a6 a5 d4 95 af c2 3b   ..~.;z.........;

    Start Time: 1680308108
    Timeout   : 7200 (sec)
    Verify return code: 21 (unable to verify the first certificate)
    Extended master secret: no
    QUIC: no
---
GET / HTTP/1.0
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ApplicationData (23)
  Length = 24
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ApplicationData (23)
  Length = 39

Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ApplicationData (23)
  Length = 24
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = ApplicationData (23)
  Length = 25
Received Record
Header:
  Version = NTLS (0x101)
  Content Type = ApplicationData (23)
  Length = 215
HTTP/1.1 200 OK
Server: Tengine/2.4.0
Date: Sat, 01 Apr 2023 00:15:15 GMT
Content-Type: application/octet-stream
Content-Length: 33
Connection: close

body NTLSv1.1:ECC-SM2-SM4-GCM-SM3Received Record
Header:
  Version = NTLS (0x101)
  Content Type = Alert (21)
  Length = 26
    Level=warning(1), description=close notify(0)

closed
Sent Record
Header:
  Version = NTLS (0x101)
  Content Type = Alert (21)
  Length = 26
    Level=warning(1), description=close notify(0)

root@5bd3e64034a7:/usr/local/nginx/logs#
```
## MySQL国密改造，基于TLS 1.3 + 商密套件

首先，需要先安装铜锁密码库，参考[构建铜锁密码库](#构建铜锁密码库)。

参考以下步骤，基于MySQL源代码，使用铜锁密码库，构建MySQL。

```bash
# 安装依赖
apt install gcc g++ cmake patchelf libncurses5-dev pkg-config

# 下载MySQL代码
wget https://downloads.mysql.com/archives/get/p/23/file/mysql-boost-8.0.32.tar.gz

tar xzvf mysql-boost-8.0.32.tar.gz

cd mysql-8.0.32

mkdir build
cd build
cmake .. -DWITH_SSL=/opt/tongsuo/ -DWITH_BOOST=/root/mysql-8.0.32/boost/

make
make install

```
配置MySQL，/etc/my.cnf如下：
```bash
[mysqld]
user 	= root
port    = 3306
basedir	= /usr/local/mysql
datadir = /usr/local/mysql/data
character-set-server = utf8
socket  = /usr/local/mysql/mysql.sock
require_secure_transport = ON
tls_ciphersuites = TLS_SM4_GCM_SM3:TLS_SM4_CCM_SM3
tls_version 	= TLSv1.3
ssl_ca		= /root/Tongsuo/test/certs/sm2-chain-ca.crt
ssl_cert	= /root/Tongsuo/test/certs/sm2-leaf.crt
ssl_key		= /root/Tongsuo/test/certs/sm2-leaf.key

[client]
port    = 3306
socket  = /usr/local/mysql/mysql.sock
default-character-set = utf8
```
启动MySQL服务器：
```bash
# 初始化数据库
/usr/local/mysql/bin/mysqld --initialize

cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql

# 启动MySQL
/etc/init.d/mysql start
```
连接MySQL：
```bash
/usr/local/mysql/bin/mysql -uroot -p --ssl-mode=required
```
查看status：
![image.png](img/handbook17.png)

## 结营作业说明

基于铜锁密码库项目或铜锁生态项目，解决客户、产品或研发中遇到的问题，可以结合个人经验、公司业务或者客户需求。以小组或个人的方式，进行软件设计和开发，不限编程语言和技术栈，最终需要提交设计PPT和代码。

选题方向参考以下，但也不限于此：

1. 发现铜锁密码库项目和铜锁生态相关项目缺少某些算法或功能，加以补充或增强；
2. 国密改造过程中发现企业内部的Web服务器（或者网关、代理、中间件等）不支持国密通信（比如国密HTTPS），基于铜锁密码库进行国密改造，以满足企业需求，比如Apache httpd等；
3. 改造技术栈中的某些模块或库，以支持国密通信，比如Python中的requests库并不支持国密HTTPS通信，考虑使用铜锁密码库改造requests库以支持国密HTTPS通信；
4. 某些编程语言或运行时还不支持国密算法或国密协议，考虑基于铜锁密码库进行改造，使其支持国密算法或国密协议，比如NodeJS、Rust等；
5. 网络通信协议国密改造，基于铜锁密码库支持更多的网络协议，比如QUIC、SSH、SFTP、LDAP等；
6. 基于铜锁密码库开发应用程序，移动端、客户端、服务端不限，或者是一个浏览器插件也可以，比如开发一个带图形界面的密码工具箱，支持SM2/SM3/SM4算法、TLCP协议等功能；

作业提交方式：

- 给出概要设计和代码。
- 概要设计可以准备1-2页PPT，讲清楚解决的问题、思路和实现。
- 代码需要提交到[https://atomgit.com/tongsuo/t-camp](https://atomgit.com/tongsuo/t-camp)，先fork该项目，再通过提交变更请求的方式提交代码，（如果没有atomgit平台账号需要先注册）。
- 以组号或者小组名字在t-camp下创建目录，将所有代码放到该目录下，避免和其他小组冲突；

fork项目：
![image.png](img/handbook18.png)

提交变更请求：
![image.png](img/handbook19.png)
