# docker部署开源密码管理器Bitwarden, 并申请免费ssl证书自动刷新永不过期

如果不需要远程共享，可以部署在本地。
要实现远程多端共享，就至少需要一台公网可访问的服务器，最好还有一个已备案的域名。
## 部署
### 下载并启动nginx
```shell
#下载
[root@ogenes01 data]#  git clone https://github.com/ogenes/docker-bitwarden.git

#进入项目
[root@ogenes01 data]# cd docker-bitwarden

#生成环境变量
[root@ogenes01 docker-bitwarden]# cp .env.example .env

#将nginx配置文件中的域名替换成自己的域名
[root@ogenes01 docker-bitwarden]# sed s/bitwarden.example.com/yourdomain/g nginx/conf.d/bitwarden.conf

#创建网络，指定子网与.env中配置一致
[root@ogenes01 docker-bitwarden]# docker network create backend --subnet=172.19.0.0/16
18f511530214374896700ad3f179fb9180227fe4e5b6ccf7e9f8ed9b8602059c
[root@ogenes01 docker-bitwarden]# docker network ls | grep backend
18f511530214   backend   bridge    local

#启动nginx
[root@ogenes01 docker-bitwarden]# docker-composer up -d nginx

```

### 启动bitwarden
```shell
[root@ogenes01 docker-bitwarden]# docker-compose up -d bitwarden
[+] Running 1/1
 ⠿ Container bitwarden  Started                   0.3s
```
### 申请SSL证书
```shell
[root@ogenes01 docker-bitwarden]# docker-compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d yourdomain
Saving debug log to /var/log/letsencrypt/letsencrypt.log
……………………
Successfully received certificate.
……………………
```

### 开启nginx的https的配置，重启nginx
```shell
#去掉注释
[root@ogenes01 docker-bitwarden]# vim nginx/conf.d/bitwarden.conf
server {
    listen 80;
    listen [::]:80;

    server_name bitwarden.example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://bitwarden.example.com$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;

    server_name bitwarden.example.com;

    ssl_certificate /etc/nginx/ssl/live/bitwarden.example.com/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/bitwarden.example.com/privkey.pem;

    location / {
        proxy_pass http://bitwarden:80/;
    }
}


#重启nginx
[root@ogenes01 docker-bitwarden]# docker-compose restart nginx
```

### 浏览器访问

![image-20230418194934116](https://img.ogenes.cn/img/2023/image-20230418194934116.png)

### 创建一个账号

![image-20230418200130938](https://img.ogenes.cn/img/2023/image-20230418200130938.png)

### 登录，设置中文
![image-20230418200318902.png](https://img.ogenes.cn/img/2023/image-20230418200318902.png)

### 更改配置，禁止注册，然后重启

尝试注册时会报错：
发生错误。
Registration not allowed or user already exists

![image-20230419150341237](https://img.ogenes.cn/img/2023/image-20230419150341237.png)

### 从chrome导出账号密码
chrome://settings/passwords
![image-20230418200627552](https://img.ogenes.cn/img/2023/image-20230418200627552.png)

### 导入到bitwarden

![image-20230418200734434](https://img.ogenes.cn/img/2023/image-20230418200734434.png)

### 下载浏览器插件
访问 https://chrome.google.com/webstore/search/bitwarden?utm_source=chrome-ntp-icon

![image-20230418195902674](https://img.ogenes.cn/img/2023/image-20230418195902674.png)

### 设置服务器url, 自动填充

![image-20230418200923646](https://img.ogenes.cn/img/2023/image-20230418200923646.png)

输入服务器url: https://bitwarden.example.com
然后登录
![image-20230418201520303](https://img.ogenes.cn/img/2023/image-20230418201520303.png)


### 自动刷新ssl证书
```shell
添加计划任务
#更新https证书
1 1 1 * * cd /data/docker-bitwarden && docker-compose run --rm certbot renew >> /dev/null 2>&1
```
