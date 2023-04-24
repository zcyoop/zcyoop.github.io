title:  'Docker Remote API的TLS认证&Portainer配置'
date:  2023-04-23
tags: Docker
categories: [容器]

# 前言

Docker 自带 API 功能，支持对 Docker 进行管理，但是默认并不启用，而且一般在没有开启加密通信的时候直接启用 API 会有安全风险。

**直接开启非加密的 Docker API 是个非常危险的行为！**

因此我们需要开启 TLS 加密来保障 Docker API 通信的安全性。通过开启 TLS 防止未经许可的客户端访问服务端节点，保障其系统安全性。

而 Docker API 也适用于在一个 Portainer 示例上来远程管理多台机器，这也是我最初使用这一功能的原因。接下来将介绍如何完成 TLS 的设置，来保障 Docker API 通信的安全性


# 开启加密通信步骤
## 第一步：创建 TLS 证书

```
#!/bin/bash
# 
# -------------------------------------------------------------
# 自动创建 Docker TLS 证书
# -------------------------------------------------------------

# 以下是配置信息
# --[BEGIN]------------------------------

PASSWORD="your code"
COUNTRY="CN"
STATE="your state"
CITY="your city"
ORGANIZATION="your org"
ORGANIZATIONAL_UNIT="your org unit"
EMAIL="your email"

# --[END]--

CODE="docker_api"
IP=`curl ip.sb -4`
COMMON_

# Generate CA key
openssl genrsa -aes256 -passout "pass:$PASSWORD" -out "ca-key-$CODE.pem" 4096
# Generate CA
openssl req -new -x509 -days 365 -key "ca-key-$CODE.pem" -sha256 -out "ca-$CODE.pem" -passin "pass:$PASSWORD" -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORGANIZATION/OU=$ORGANIZATIONAL_UNIT/CN=$COMMON_NAME/emailAddress=$EMAIL"
# Generate Server key
openssl genrsa -out "server-key-$CODE.pem" 4096

# Generate Server Certs.
openssl req -subj "/CN=$COMMON_NAME" -sha256 -new -key "server-key-$CODE.pem" -out server.csr

echo "subjectAltName = IP:$IP,IP:127.0.0.1" >> extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "server-cert-$CODE.pem" -extfile extfile.cnf


# Generate Client Certs.
rm -f extfile.cnf

openssl genrsa -out "key-$CODE.pem" 4096
openssl req -subj '/CN=client' -new -key "key-$CODE.pem" -out client.csr
echo extendedKeyUsage = clientAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "cert-$CODE.pem" -extfile extfile.cnf

rm -vf client.csr server.csr

chmod -v 0400 "ca-key-$CODE.pem" "key-$CODE.pem" "server-key-$CODE.pem"
chmod -v 0444 "ca-$CODE.pem" "server-cert-$CODE.pem" "cert-$CODE.pem"

# 打包客户端证书
mkdir -p "tls-client-certs-$CODE"
cp -f "ca-$CODE.pem" "cert-$CODE.pem" "key-$CODE.pem" "tls-client-certs-$CODE/"
cd "tls-client-certs-$CODE"
tar zcf "tls-client-certs-$CODE.tar.gz" *
mv "tls-client-certs-$CODE.tar.gz" ../
cd ..
rm -rf "tls-client-certs-$CODE"

# 拷贝服务端证书
mkdir -p /srv/certs.d
cp "ca-$CODE.pem" "server-cert-$CODE.pem" "server-key-$CODE.pem" "tls-client-certs-$CODE.tar.gz" /srv/certs.d/
```

修改上面的配置信息，将上述脚本保存为一个 `.sh` 文件，并在命令行执行 `bash xxx.sh` 以运行该脚本。运行后会在当前目录生成服务端和客户端证书信息，同时会在将在 `/srv/certs.d` 目录下生成证书，其中包括 `tls-client-certs-docker_api.tar.gz` ，该文件保存着客户端访问 Docker API 所需的证书，请下载下来并安全和妥善地保存。

## 第二步：开启 Docker API

首先打开终端并执行以下命令：

```
vim /lib/systemd/system/docker.service
```

在打开的 Docker 服务文件中查找 ExecStart 行

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picgo20230423174126.png)

在 ExecStart 行的后面添加以下选项：

```
-H=tcp://0.0.0.0:2376 --tlsverify --tlscacert=/srv/certs.d/ca-docker_api.pem --tlscert=/srv/certs.d/server-cert-docker_api.pem --tlskey=/srv/certs.d/server-key-docker_api.pem
```

最后，执行以下命令重新加载服务并重启 Docker：

```
systemctl daemon-reload && service docker restart
```

## 第三步：验证

此时，我们需要验证 Docker API 是否能够访问，且是否只能通过加密访问。

首先执行 `docker -H=127.0.0.1:2376 info`，一般来说会返回：

```
Client:
Context:    default
Debug Mode: false
Plugins:
app: Docker App (Docker Inc., v0.9.1-beta3)
buildx: Docker Buildx (Docker Inc., v0.9.1-docker)
Server:
ERROR: Error response from daemon: Client sent an HTTP request to an HTTPS server.
errors pretty printing info
```

这是因为没有通过 tls 去访问，此时改用 `docker -H=127.0.0.1:2376 --tlsverify info`，会出现下面的错误：

```
unable to resolve docker endpoint: open /root/.docker/ca.pem: no such file or directory
```

这是由于目前没有在对应的用户文件夹下配置证书，我们可以执行以下命令：

```
mkdir ~/.docker && \
tar -zxvf /srv/certs.d/tls-client-certs-docker_api.tar.gz -C ~/.docker && \
mv ~/.docker/ca-docker_api.pem ~/.docker/ca.pem && \
mv ~/.docker/cert-docker_api.pem ~/.docker/cert.pem && \
mv ~/.docker/key-docker_api.pem ~/.docker/key.pem
```

完成后再执行一遍 `docker -H=127.0.0.1:2376 --tlsverify info` 即可获取信息了，至此验证完成。

## 第四步：配置 Portainer

- 打开 Portainer，进入 Environments，点击右上角的 Add environment 添加节点，并在随后的向导中选择 `Docker`

![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picgo20230423174458.png)



- 在`Docker environment`中选择API模式并输入`Name`和`Docker API URL`


- 将 Environment URL 或者 Docker API URL 中的端口从 2375 改为 2376。然后将 TLS 的 Switch 打开，即可看到相关的 TLS 配置项，我们在第一步的时候下载了一份文件，解压后根据名称匹配即可。
![](https://blog-1253533258.cos.ap-shanghai.myqcloud.com/picgo20230423175344.png)



# 参考

- [如何开启 Docker Remote API 的 TLS 认证，并在 Portainer 上进行配置](https://www.xukecheng.tech/how-to-enable-tls-authentication-for-docker-remote-api)
