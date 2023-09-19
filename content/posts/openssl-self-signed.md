---
title: "OpenSSL 生成自签名证书"
date: 2023-09-19T23:34:26+08:00
archives: 
    - 2023
tags:
    - 网络
    - 安全
image: /images/ssl-certificate.jpeg
leading: false
draft: false
---

免费证书有效期一般较短，当服务器较多时，每个服务器都要安装脚本更新证书，管理起来比较麻烦。而合理使用自签名证书（比如测试或者非公开环境）也是不错的替代选择。

## 生成 CA 根证书

**CA (Certificate Authority)** 被称为**证书授权中心**，是数字证书发放和管理的机构。

> 根证书是 CA 认证中心给自己颁发的证书，是信任链的起始点。安装根证书意味着对这个 CA 认证中心的信任。

### 生成私钥

可以选择以下任何一种算法：

1. **椭圆曲线（ECC）**：

    ```shell
    openssl ecparam -genkey -name secp384r1 -out ca.key
    ```

    若需要加密私钥：

    ```shell
    openssl ecparam -genkey -name secp384r1 | openssl ec -aes256 -out ca.key
    ```

    其中 `-name` 参数指定使用的曲线，曲线名称可通过以下命令查看：

    ```shell
    openssl ecparam -list_curves
    ```

2. **RSA**：

    ```shell
    openssl genrsa -out ca.key 4096
    ```

    若需要加密私钥：

    ```shell
    openssl genrsa -des3 -out ca.key 4096
    ```

### 生成根证书

使用上面的私钥生成一个证书，证书会包含一些组织信息和公钥。为了一劳永逸有效期自然越长越好，本例设置为 73000 天（20 年）。

如果是 RSA，可以使用 `-sha256` 签名。

```shell
openssl req -key ca.key -new -x509 -days 73000 -sha384 -subj "/C=CN/ST=Guangdong Province/L=Shenzhen/O=Shenzhen XXX Co., Ltd/CN=XXX Root CA" -out ca.crt
```

一些说明：

- `-x509` 参数表明生成**自签名证书**，若没有传该参数，则生成一个**证书请求文件**（下文有例子）。

- `-subj` 参数填写了一些组织信息，若没有传该参数，生成时程序会进行询问。以下是信息说明：

```
国家（Country Name, C）：CN
州或省（State or Province Name, ST）：Guangdong Province
城市（Locality Name, L）：Shenzhen
组织（Organization Name, O）：Shenzhen XXX Co., Ltd
单位（Organizational Unit Name, OU）：R & D Center
公用名（Common Name, CN）：XXX Root CA
邮箱（Email Address, emailAddress）：admin@example.com
```

## 生成网站证书

下面以生成一个泛域名 `*.example.com` 证书为例。

### 生成私钥

此步骤和 CA 证书私钥生成步骤相同，只是参数可能会修改一下。

相对于根证书，客户端证书的密钥长度可以稍微小一点，但建议 RSA 至少 2048 位，ECC 至少 256 位。

同样可以选择以下任何一种算法：

1. **椭圆曲线（ECC）**：

    例如使用 `P-256` 曲线：

    ```shell
    openssl ecparam -genkey -name prime256v1 -out _.example.com.key
    ```

2. **RSA**：

    ```shell
    openssl genrsa -out _.example.com.key 2048
    ```

### 生成证书请求文件

命令和上面根证书的生成差不多，注意这里是没有 `-x509` 参数的，表明生成一个证书请求文件。

```shell
openssl req -key _.example.com.key -new -subj "/CN=*.example.com" -out _.example.com.csr
```

### 使用 CA 根证书对网站证书签名

首先新建一个文件 `v3.ext` 填写证书的一些扩展信息：

```ini
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid, issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature
extendedKeyUsage = serverAuth, clientAuth
certificatePolicies = 2.23.140.1.2.1
subjectAltName = @alt_names

[alt_names]
DNS.1 = *.example.com
```

其中 `alt_names` 可以填写多个域名和 IP，例如 `DNS.2 = alt.example.com`，`IP.1 = 127.0.0.1`。

下面命令利用 CA 根证书对网站证书进行签名，生成最终的网站证书。

```shell
openssl x509 -req -CA ca.crt -CAkey ca.key -days 65700 -sha384 -extfile v3.ext -in _.example.com.csr -out _.example.com.crt
```

如果是 RSA，可以使用 `-sha256` 签名。

### 生成全链证书

证书以证书链的形式存在，只有当整个证书链上的证书都有效时，才会认定当前证书是合法的。 

1. 最上层为**根证书**，也就是通常所说的 CA，用于颁发证书。
2. 中间一层为**中间证书**，是二级CA，这一层可以继续划分为多层，用来帮助 CA 给用户颁发证书。
3. 最下层为**用户证书**，对应是每个网站使用的 SSL 证书。

**全链证书**，顾名思义就是把整个证书链的证书都放到同一个证书文件中，具体操作如下：

```shell
cat _.example.com.crt <(echo) ca.crt > _.example.com.fullchain.crt
```

注意证书文件的顺序，从下层往上层依次添加证书文件。

最后，可以把证书文件 `_.example.com.fullchain.crt` 和密钥文件 `_.example.com.key` 部署到 Web 服务器。
