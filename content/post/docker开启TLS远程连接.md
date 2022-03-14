---
title: "Docker开启TLS远程连接"
date: 2021-05-14T10:29:42+08:00
Description: ""
Tags: ["Docker","容器"]
Categories: ["Docker"]
DisableComments: false
---

## Docker远程连接

Docker是C/S架构，通过docker的客户端连接到docker daemon，然后由docker daemon在宿主机上执行构建镜像，运行容器等指令。

Docker支持多种方式将docker客户端连接到server。如果docker客户端和server都在同一台宿主机上，那么可以直接使用Unix域套接字文件`/var/run/docker.sock`连接，我们直接使用docker命令时实际上就是采用这种方式进行连接的。

Docker也支持远程通过socket进行连接，可以选择开启或者不开启TLS，为了安全我们大多数时候还是采用开启TLS的方式进行远程连接。

## 启动远程连接的过程

首先需要保证宿主机上docker daemon已安装，安装过程可查看[官网](https://docs.docker.com/engine/install/ubuntu/)

### 制作证书

首先需要制作证书，包括服务端证书，和客户端证书。

在docker daemon的宿主机上按步骤执行下列命令：

1. 创建根证书RSA私钥
   命令输入后会提示输入密码，输入两次后会生成**ca-key.pem**文件。

   ```bash
   $ openssl genrsa -aes256 -out ca-key.pem 4096
   ```

2. 用这个RSA密钥创建CA证书

   自己给自己签发证书，也可以交给第三方机构签发。这里我们自己签发。`-days`参数指定多少天后过期，这里设置1000天。这一步会生成**ca.pem**文件，就是CA证书。

   ```bash
   $ openssl req -new -x509 -days 1000 -key ca-key.pem -sha256 -subj "/CN=*" -out ca.pem
   ```

3. 创建服务端私钥

   这一步生成的**server-key.pem**就是服务端私钥。

   ```bash
   $ openssl req -subj "/CN=*" -sha256 -new -key server-key.pem -out server.csr
   ```

4. 生成服务端证书签名请求csr文件

   csr（certificate signing request），里面包含公钥和服务端信息。这一步会生成**server.csr**文件，就是服务端证书。

   ```bash
   $ openssl req -subj "/CN=*" -sha256 -new -key server-key.pem -out server.csr
   ```

5. 生成签名过的服务端证书

   服务端证书需要签名，这一步生成的**server-cert.pem**文件就是签名过的服务端证书

   ```bash
   $ openssl x509 -req -days 1000 -sha256 -in server.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out server-cert.pem
   ```

6. 生成客户端私钥

   接下来我们生成客户端的私钥**key.pem**

   ```bash
   $ openssl genrsa -out key.pem 4096
   ```

7. 生成客户端证书签名请求

   生成的**client.csr**文件就是客户端签名请求

   ```bash
   $ openssl req -subj "/CN=client" -new -key key.pem -out client.csr
   ```

8. 生成**exfile.cnf**配置文件

   ```bash
   $ echo extendedKeyUsage=clientAuth > extfile.cnf
   ```

9. 生成签名过的客户端证书

   期间要输入密码，生成的**cert.pem**就是签名过的客户端证书

   ```bash
   $ openssl x509 -req -days 1000 -sha256 -in client.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out cert.pem -extfile extfile.cnf
   ```

10. 最后删除多余的中间文件

    ```bash
    $ rm -rf ca.srl client.csr extfile.cnf server.csr
    ```

    剩下的文件都是我们需要的：

    - ca.pem：CA机构证书
    - ca-key.pem：根证书RSA私钥
    - cert.pem：客户端证书
    - key.pem：客户端私钥
    - server-cert.pem：服务端证书
    - server-key.pem：服务端私钥

### 服务端设置

证书生成完成后，需要对docker daemon进行设置让它允许远程TLS访问。

1. 首先打开打开`/lib/systemd/system/docker.service`文件，找到这行内容：

   ```
   ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
   ```

   把这行替换为以下格式：

   ```
   ExecStart=/usr/bin/dockerd --tlsverify --tlscacert=/ca.pem --tlscert=/server-cert.pem --tlskey=/server-key.pem -H tcp://0.0.0.0:2376 -H unix:///var/run/docker.sock
   ```

   注意几个参数指定的文件的位置：

   - --tlscacert：CA根证书
   - --tlscert：服务端证书
   - --tlskey：服务端私钥

   `-H`参数指定了访问方式，这里设置了开启远程连接，端口为2376，并且依然支持Unix域套接字连接(即本地直接连接)

2. 然后如果我们的机器没有放通2376端口，需要放通一下：

   ```bash
   $ sudo iptables -I INPUT -p tcp --dport 2376-j ACCEPT
   ```

3. 最后重启docker daemon让配置生效

   ```bash
   $ systemctl daemon-reload && systemctl restart docker
   ```

   没报错就说明配置成功了

### 客户端远程连接

完成上述的配置后，服务端的docker daemon就支持TLS远程连接了。客户端只需要在连接时指定一些参数就可以连接到远程的docker daemon上。

1. 首先把在服务端创建的三个文件**ca.pem**，**cert.pem**和**key.pem**复制到客户端机器上。

2. 然后我们给服务端ip设置个域名

   在`/etc/hosts`文件中添加一行，前面的服务端ip，后面是它的域名。

   ```
   192.168.121.138 docker-daemon
   ```

3. 然后就可以进行连接了。

   指定客户端的三个文件(这里假设都在`/root/work/`目录下)，通过-H指定服务端的地址

   ```bash
   $ docker --tlsverify --tlscacert=/root/work/ca.pem --tlscert=/root/work/cert.pem --tlskey=/root/work/key.pem -H tcp://docker-daemon:2376 version
   ```

   ![](/images/docker_remote/version.png)

   可以看到客户端和服务端的版本，其他命令也都是可以的。

   还可以把几个证书文件拷贝到`~/.docker/`目录下，这样在连接时就可以不指定--tlscacert、--tlscert和--tlskey这几个参数了，docker client会自动加载这几个文件的。

   ```bash
   $ docker --tlsverify -H=tcp://docker-daemon:2376 version
   ```

   

