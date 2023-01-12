
我在我的服务器上安装了 Proxmox，我在使用的时通过默认的 8006 端口访问 Web 界面，要求必须使用 HTTPS 进行访问，但是我并没有打算在这台机器是配置安全证数，所以有了以下操作。

## 第一步：自签名证书

### 1. 对现有证数进行转换

其实在安装 Proxmox 的时候，会自动地创建一个本地证数，位置在 `/etc/pve/nodes/pve/` 目录下 `pve-ssl.key` `pve-ssl.pem` ,我们只需要复制出一份新的文件将后缀名 `.pem` 改成 `.crt` 即可
``` shell
cp pve-ssl.pem pve-ssl.crt
```

> 详细证数类型请看：
> [跳转1](https://docs.azure.cn/zh-cn/articles/azure-operations-guide/application-gateway/aog-application-gateway-howto-create-self-signed-cert-via-openssl)
> [跳转2](https://learn.microsoft.com/zh-cn/azure/application-gateway/self-signed-certificates)

### 2. 创建新的本地证书
通过 openssl 可以创建出一个本地证书，我们最终需要的是 .crt 的证数，通过下列命令可以创建出一个 .crt 的证数。
``` shell
openssl req -new -x509 -newkey rsa:4096 -keyout pve-ssl.key -out pve-ssl.crt
```
我们还需要一份后缀 `.pem` 的证数文件用于替换 Proxmox 的默认证数文件
``` shell
cp pve-ssl.crt pve-ssl.pem
```
替换 Proxmox 的默认证数
``` shell
rm -f /etc/pve/nodes/pve/pve-ssl.*
cp pve-ssl.* /etc/pve/nodes/pve/
```

## 第二步：为本地系统添加证数信任

我们将上一步的到的 `.crt` 证书文件拷贝一份到 `/usr/local/share/ca-certificates/` 目录
然后我们运行下列命令即可，输出的提示消息中应当包含 `1 added` 字样
``` shell
update-ca-certificates
```

这时候我们重启系统，重启完成之后我们可以使用下列命令进行测试

``` shell
curl -v https://localhost:8006
```

应当输出 Proxmox 登陆页面的 HTML 文档，而不会有任何异常

## 第三步：使用 Nginx 进行反向代理

首先我们安装 `Nginx`
``` shell
sudo apt install nginx
```

然后我们在 `/etc/nginx/conf.d/` 目录下添加 Nginx 的配置文件，或者是修改他的默认配置文件也可

> 默认的配置有可能在 `/etc/nginx/sites-enabled/default` 和 `/etc/nginx/nginx.conf` 这两个文件的任意一个，具体真实情况需要看使用的 Nginx 版本

```
server {
        listen 8005 default_server;
        listen [::]:8005 default_server;

        server_name _;

        location / {
                proxy_pass https://localhost:8006;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection keep-alive;
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
        }
}
```

然后重新加载 Nginx 配置文件或者重新启动服务
``` shell
nginx -s reload
# OR
systemctl restart nginx
```

这时候我们使用 `HTTP` 与 `8005` 端口进行访问

``` shell
curl -v http://localhost:8005
```

这时候的输出与使用 `HTTPS` 和 `8006` 的结果是一致的，这样我们就绕开了 Proxmox 必须使用 TSL 的限制


> 我使用的是 `Proxmox 7.2` 版本，这个版本是基于 `Debian 11` 进行开发的