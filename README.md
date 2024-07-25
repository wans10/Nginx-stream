# Nginx-stream
当我们手上的vps越来越多，就可以将他们通过负载均衡统一起来，实现自动切换0宕机。

无论是SS、SSR还是V2ray,又或者是其他，都是一个思路，下面演示把它们统称为代理。

这里演示我们有4个装有代理的小鸡，分别为小鸡A、B、C、D，ip和端口分别为：1.1.1.1:10000、2.2.2.2:20000、3.3.3.3:30000、4.4.4.4:40000；准备一台国内服务器搭建nginx做负载均衡用，取名为小鸡E。

## 安装nginx

在小鸡E上安装nginx，推荐使用Ubuntu/Debian系统。

```bash
Ubuntu/Debian
apt-get update
apt-get install nginx -y
service nginx start
```

安装完毕后，在浏览器中输入小鸡E的ip地址，能打开网站即为成功。否则检查nginx是否启动，阿里腾讯等服务商需要额外检查防火墙是否开放80端口。

找到了 stream 模块需要的包，接下来安装就好了：

```bash
sudo apt search nginx
sudo apt install libnginx-mod-stream
```

## 配置负载均衡

创建并打开一个自定义的配置文件

```bash
mkdir -p /etc/nginx/tcpconf.d
vi /etc/nginx/tcpconf.d/ssrproxy.conf
```

写入如下配置：

```bash
stream {
    upstream group1 {
        server 1.1.1.1:10000;
        server 2.2.2.2:20000;
    }
    server {
        listen 10000;
        listen 10000 udp;
        proxy_pass group1;
    }
    upstream group2 {
        server 3.3.3.3:30000;
        server 4.4.4.4:40000;
    }
    server {
        listen 20000;
        listen 20000 udp;
        proxy_pass group2;
    }
}
```

这里一共分成了两组：group1和group2。使用小鸡E的10000端口对应group1，用来转发流量到小鸡A（1.1.1.1）或小鸡B（2.2.2.2），小鸡E的20000端口对应group2，用来转发流量到小鸡C（3.3.3.3）或小鸡D（4.4.4.4）。每个组都有一个或多个流量转发的对象，比如group1里有1.1.1.1和2.2.2.2，不是说流量同时转发到这两个服务器，而是通过nginx的负载均衡，自动轮询并转发流量到合适的服务器。

之后我们找到nginx的主配置文件，将上面自定义配置加载进nginx里。主配置文件一般在 /etc/nginx/nginx.conf 或者 /etc/nginx/conf/nginx.conf 里。

```bash
echo "include /etc/nginx/tcpconf.d/*.conf;" >> /etc/nginx/nginx.conf
```

重启一下nginx使配置生效

```bash
service nginx restart
```
