*使用x-ui判断配置的证书是否有效*

## 1. 安装x-ui

使用一键脚本安装
```
bash <(curl -Ls https://raw.githubusercontent.com/FranzKafkaYu/x-ui/956bf85bbac978d56c0e319c5fac2d6db7df9564/install.sh) 0.3.4.4
```

## 2. 申请CF证书
https://65536.io/2020/03/607.html

## 3. 配置证书
随便找个地方存储证书

## 4. 安装Nginx
`apt-get install nginx`

## 5. 配置Nginx
默认路径 /etc/nginx/nginx.conf
配置中可以include其他配置, 在http 内增加include /etc/nginx/conf.d/*.conf;
```
# ssl配置
server {
    listen 443 ssl default_server;
    server_name bwg.maxchan.us.kg;
    ssl_certificate /etc/nginx/cert/public.pem; #指定上面保存的证书
    ssl_certificate_key /etc/nginx/cert/private.key; #指定上面的私钥
}
server
    {   
    listen 80;
    server_name bwg.maxchan.us.kg;  #注意将域名替换成你所使用的实际域名
    access_log  /var/log/nginx/bwg.maxchan.us.kg.access.log;
    location / {
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Range $http_range;
        proxy_set_header If-Range $http_if_rang;
        proxy_redirect off;
        proxy_pass http://127.0.0.1:1017; #注意将ip和端口号根据实际情况替换

        # the max size of file to upload
        client_max_body_size 20000m;
    }
}
```