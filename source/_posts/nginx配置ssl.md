---
title: Nginx 配置 SSL
date: 2019-08-31 10:35:31
categories: 
- Nginx
tags:
- nginx
- devOps
---

> 域名貌似备案成功了，解析到腾讯云服务器ip后，发现直接访问 http协议 是可以的 换成 https 不行了。应该是没有配置SSL导致的。

## 什么是HTTPS
https 全称：Hyper Text Transfer Protocol over Secure Socket Layer，是http的安全版。即http下加入SSL协议层，因此https的安全基础就是SSL，所以加密内容需要SSL。

## SSL原理
- 浏览器发送一个https的请求给服务器；
- 服务器要有一套数字证书，可以自己制作，也可以向组织申请，区别就是自己颁发的证书需要客户端验证通过，才可以继续访问，而使用受信任的公司申请的证书则不会弹出提示页面，这套证书其实就是一对公钥和私钥；
- 服务器会把公钥传输给客户端；
- 客户端（浏览器）收到公钥后，会验证其是否合法有效，无效会有警告提醒，有效则会生成一串随机数，并用收到的公钥加密；
- 客户端把加密后的随机字符串传输给服务器；
- 服务器收到加密随机字符串后，先用私钥解密（公钥加密，私钥解密），获取到这一串随机数后，再用这串随机字符串加密传输的数据（该加密为对称加密，所谓对称加密，就是将数据和私钥也就是这个随机字符串>通过某种算法混合在一起，这样除非知道私钥，否则无法获取数据内容）；
- 服务器把加密后的数据传输给客户端；
- 客户端收到数据后，再用自己的私钥也就是那个随机字符串解密；

## 配置 with-http_ssl_module 模块
首先需要申请一个证书，可以申请一个免费的。
参考下面的 `其他方式` 部分可以直接在云服务厂商可视化界面申请并下载~
然后会得到nginx版本证书，一个公钥（证书），一个私钥，将其上传到服务器。

先确认`nginx`安装时已编译`http_ssl`模块，也就是执行`nginx -V`命令查看是否存在 `--with-http_ssl_module`。如果没有，则需要重新编译nginx将该模块加入。
若有的话此步骤跳过
若ssl模块没有先编译一下

```bash
cd /usr/local/src/nginx-1.12.1/  #nginx版本号可能不同哟

./configure --help | grep -i ssl
  --with-http_ssl_module 
  
./configure --prefix=/usr/local/nginx --with-http_ssl_module

```

## 生成 ssl 秘钥对
```bash
cd /usr/local/nginx/conf

yum install -y openssl
```

### 使用RSA算法生产key
```bash
openssl genrsa -des3 -out tmp.key 2048
# 生成一个rsa类型的密钥，且长度为2048，但是我们有发现，让我们设置密码，如果每次有人访问我们的站点，都需要输入密码，太麻烦了
```
所以我们要转换私钥，取消密码（其实tmp.key与zhdy.key密钥内容是一样的，只不过一个有密码一个没有）：
```bash
openssl rsa -in tmp.key -out zhdy.key 

rm -f tmp.key
```
### 创建证书申请
```
openssl req -new -key zhdy.key -out zhdy.csr
```

### 创建自签名的证书
```bash
openssl x509 -req -days 365 -in zhdy.csr -signkey zhdy.key -out zhdy.crt
```

## Nginx 配置 ssl
```bash
server
{
    listen 443;
    server_name huguobo.site; # 换成你自己的域名服务哟
    index index.html index.php;
    root /path/to/your/file 
    ssl on;
    ssl_certificate zhdy.crt;
    ssl_certificate_key zhdy.key;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
}

# ssl on; 代表着打开ssl
#ssl_certificate zhdy.crt; 公钥
# ssl_certificate_key zhdy.key; 私钥
# ssl_protocols TLSv1 TLSv1.1 TLSv1.2; 三种不同的协议
```

## 测试
```bash
nginx -t
nginx restart
```

由于已经设置了443，所以我们使用curl -x 是不生效的。
我们修改下/etc/hosts为: `yourip www.aaa.com`

```bash
curl https://haha.com
```
报错显示为“此证书非安全证书”，但是ssl是已经成功配置了

## 其他方式（更快捷）
后来发现各个厂商也提供申请证书的入口，也提供 DNS 和 文件验证两种配置,大家也可以自行参考配置
[腾讯云ssl控制台](https://console.cloud.tencent.com/ssl)
[nginx配置文档](https://cloud.tencent.com/document/product/400/35244)

这种从供应商处可以直接下载到证书，然后nginx的配置秘钥和证书都在 nginx 文件夹下

从客户端传送文件到云主机（MAC）
```bash
scp  ./2_huguobo.site.key ./1_huguobo.site_bundle.crt  root@118.24.215.220:/usr/local/nginx/conf
```
登录云主机 nginx 配置
```bash
# Settings for a TLS enabled server.

    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  huguobo.site;

        ssl on;
        ssl_certificate "/usr/local/nginx/conf/1_huguobo.site_bundle.crt";
        ssl_certificate_key "/usr/local/nginx/conf/2_huguobo.site.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE; #请按照这个套件配置，配置加密套件，写法遵循 openssl 标准。

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
           root /home/hexoBlog; # your path
           index index.html;
        }

        error_page 404 /404.html;
            location = /40x.html {
        }
        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
   }

```
## 强制HTTPS
我没有做强制https。但是需要的话，改下 nginx 配置文件即可
http 默认是`80`端口 ，https 是`443`端口
```bash
server{
      listen 80;    #表示监听80端口
      server_name huguobo.site
      location / {    #将80端口强制转为https
          rewrite (.*) https://huguobo.site$1 permanent;
      }
}
```
