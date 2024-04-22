---
sidebar_position: 1
---
# 欧拉操作系统上安装铜锁 RPM 包

## 前言

铜锁和欧拉开源操作系统作为开放原子开源基金会的2个开源项目，目前都处于孵化期。铜锁项目作为基础密码库，希望能够助力开源操作系统，包括欧拉系统等，为用户提供密码学基础能力，解决密码合规等问题。
为方便在欧拉操作系统上的用户使用铜锁密码库，目前已经基于Tongsuo-8.3.3源代码构建RPM包，发布到openEuler软件包仓库[https://search.oepkgs.net/](https://search.oepkgs.net/)。支持的架构包括x86_64和arm64，系统版本支持20.03-LTS-SP1、20.03-LTS-SP2和20.03-LTS-SP3。
下面以华为云弹性云服务器为例，在openEuler 20.03系统上，安装铜锁并演示国密功能。

## 实战华为云服务器+欧拉+铜锁

购买弹性云服务器时，镜像选择公共镜像里面的openEuler，版本为openEuler 20.03，如下图所示。
![image.png](img/image.png)
登录云服务器后，安装铜锁，步骤如下。

1. 添加源
    ```bash
    dnf config-manager --add-repo https://repo.oepkgs.net/openeuler/rpm/openEuler-20.03-LTS-SP1/extras/x86_64/
    ```

2. 更新源索引
    ```bash
    dnf clean all && dnf makecache
    ```

3. 安装铜锁软件包
    ```bash
    dnf install tongsuo --nogpgcheck
    ```
    铜锁安装目录为`/opt/tongsuo/`，安装成功后，可以查看铜锁版本号：
    ```bash
    /opt/tongsuo/bin/openssl version
    ```

