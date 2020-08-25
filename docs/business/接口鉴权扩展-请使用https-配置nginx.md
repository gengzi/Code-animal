[TOC]

## 目标

* 了解nginx配置https

  参考：[Nginx 配置 HTTPS 服务器](https://aotu.io/notes/2016/08/16/nginx-https/index.html)

  [开启Nginx的SSL模块](https://www.cnblogs.com/ghjbk/p/6744131.html)

  [nginx-1：生产级别nginx高性能配置](https://cloud.tencent.com/developer/article/1456381)

  [nginx 配置](https://github.com/hepyu/k8s-app-config/blob/master/yaml/min-cluster-allinone/nginx/nginx.conf.desc)

  [Nginx https 双向认证](https://www.cnblogs.com/yelao/p/9486882.html )

##  安装配置nginx

环境： ubuntu 18.0.4

这是个比较简单的安装步骤

```shell
-- 解压nginx :   tar -zxvf nginx-1.14.2.tar.gz
-- 进入解压后的目录：   cd nginx-1.14.2
-- 执行nginx要安装的位置：   ./configure --prefix=/usr/local/nginx-1.14.2 --with-http_stub_status_module --with-http_ssl_module
-- 在当前目录执行： make
-- 进入nginx 的安装目录（ /usr/local/nginx-1.14.2 ），执行：make install
-- 进入sbin 执行: ./nginx  启动
-- 进入conf目录配置nginx，默认80端口：
-- 测试nginx能否访问成功,或者使用浏览器访问： curl localhost
--  /usr/local/nginx/sbin/nginx -t测试配置文件修改是否正常
 
--  重新加载配置 ./nginx -s reload
```

注意    ./configure 这个配置，  --with-http_ssl_module 追加上这个配置，nginx才能开启 https。

如果不配置 nginx 开启https 会报如下错误

```
nginx: [emerg] SSL_CTX_load_verify_locations("/usr/local/nginx/cert/ca.cer") failed (SSL: error:0B084088:x509 certificate routines:X509_load_cert_crl_file:no certificate or crl found)
```

如果是已经存在的nginx，也可以修改。参考：[开始Nginx的SSL模块](https://www.cnblogs.com/ghjbk/p/6744131.html)

### 配置nginx.conf 文件

先将证书文件私钥都拷贝至服务器某个目录

```shell
# 进入安装目录 /usr/local/nginx-1.14.2
# 再进入 conf 目录，修改nginx.conf 配置
 
    server {
        listen       443 ssl;
        server_name  ttt.com;
        ssl                  on; 
        ssl_certificate      /usr/local/nginx/cert/server-cert.crt;  #server证书公钥
        ssl_certificate_key  /usr/local/nginx/cert/server-key.key; #server私钥
        ssl_client_certificate /usr/local/nginx/cert/newca.cer;  #根级证书公钥，用于验证各个二级client
        ssl_verify_client on;  #开启客户端证书验证 
 
        location / {
            root   html;
            index  index.html index.htm;
        }
    }
 
 
```

注意几点：

证书的文件格式，在上篇中生成的证书文件和私钥文件格式都是pem，可以通过重命名的方式，修改为 crt 和 key， cer 即可。

证书存放的目录，需要有权限访问到。

启动nginx 验证是否成功

https://localhost

## 一些问题和解决方法

当测试nginx 访问spring boot 的工程，发现swagger 的页面展示不出来了。

https://localhost/swagger-ui.html

参考：[解决nginx访问不到swagger](https://blog.csdn.net/Javamine/article/details/87433289)

[用nginx最代理请求总是 required is not finished yet](https://blog.csdn.net/saygood999/article/details/73277011)

解决办法：修改nginx 的配置

```shell
 
               location / {
                 proxy_redirect off;
                 # proxy_set_header Host $host;
                   proxy_set_header Host $host:8088; #添加:$server_port
                 proxy_set_header X-Real-IP $remote_addr;
                 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                 proxy_pass  http://nodes;
                }
 
               location /swagger-ui.html{
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header X-Forwarded-Host $host;
                    proxy_set_header X-Forwarded-Port $server_port;
                    proxy_pass http://nodes/swagger-ui.html;
                }
 
               location /swagger-resources {
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header X-Forwarded-Host $host;
                    proxy_set_header X-Forwarded-Port $server_port;
                    proxy_pass http://nodes/swagger-resources;
                }
 
               location /v2/api-docs {
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header X-Forwarded-Host $host;
                    proxy_set_header X-Forwarded-Port $server_port;
                    proxy_pass http://nodes/v2/api-docs;
                }
 
                location /webjars{
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header X-Forwarded-Host $host;
                    proxy_set_header X-Forwarded-Port $server_port;
                    proxy_pass http://nodes/webjars/;
                    client_max_body_size 50m;
                    client_body_buffer_size 256k;
                    proxy_connect_timeout 1;
                    proxy_send_timeout 30;
                    proxy_read_timeout 60;
                    proxy_buffer_size 256k;
                    proxy_buffers 4 256k;
                    proxy_busy_buffers_size 256k;
                    proxy_temp_file_write_size 256k;
                    proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
                    proxy_max_temp_file_size 128m;
                    add_header Cache-Control no-store;
                }
 
                location /webjars/springfox-swagger-ui{
                    # 请求报文大小
                    client_max_body_size 50m;
                    # 设定了request body的缓冲大小
                    client_body_buffer_size 256k;
                    add_header Cache-Control no-store;
                    #  与后端建立连接的超时时间,单位秒，默认值60秒。
                    proxy_connect_timeout 1;
                    proxy_send_timeout 30;
                    proxy_read_timeout 60;
                    proxy_buffer_size 256k;
                    proxy_buffers 4 256k;
                    proxy_busy_buffers_size 256k;
                    proxy_temp_file_write_size 256k;
                    proxy_next_upstream error timeout invalid_header http_500 http_503 http_404;
                    proxy_max_temp_file_size 128m;
                    # 即允许重新定义或添加字段传递给代理服务器的请求头。
                    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                    proxy_set_header X-Forwarded-Proto $scheme;
                    proxy_set_header X-Forwarded-Host $host;
                    proxy_set_header X-Forwarded-Port $server_port;
                    proxy_pass http://nodes/webjars/springfox-swagger-ui;
                }
 
```

具体可参考：[codecopy](https://github.com/gengzi/codecopy/tree/master/src/main/resources/nginx)  nginx_https.conf 文件

## nginx 配置优化

参考：[nginx.conf.desc](https://github.com/hepyu/k8s-app-config/blob/master/yaml/min-cluster-allinone/nginx/nginx.conf.desc) 可以参阅一下

 

 