# SSL 证书存放目录

请在此目录下保存您的 SSL 证书文件：
- `miniclassisland.com.crt` (证书文件)
- `miniclassisland.com.key` (私钥文件)

Nginx 配置文件已设置为直接读取此目录下的证书：
- `ssl_certificate     D:/.dev/miniclassisland/nginx/tls-cert/miniclassisland.com.crt;`
- `ssl_certificate_key D:/.dev/miniclassisland/nginx/tls-cert/miniclassisland.com.key;`

在部署到 Linux 等其他环境时，请根据实际证书路径修改 Nginx 配置，或在此处放置对应的证书软链接/副本。
