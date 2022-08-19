# SJTU 选课社区（jCourse）部署指南

SJTU 选课社区（jCourse）是前后端分离的Web应用。

在选课社区实际运行中，除了前后端以外，还有HTTP Server、缓存、数据库、监控等组件。涉及到的组件多，配置多，因此需要统一的文档记录其部署和配置流程，以免后来者浪费太多时间摸索。

## 0. 配置系统和依赖

准备一台 Linux 服务器（以Ubuntu为例），安装 Nginx ， Docker 和 Docker Compose。

## 1. 下载代码

选一个 jCourse 目录作为项目文件夹，下载前后端代码。
```
git clone https://github.com/SJTU-jCourse/jcourse frontend
git clone https://github.com/SJTU-jCourse/jcourse_api backend
```

## 2. 配置 Docker Compose

配置 docker-compose.yml 文件，内容如下：
```yaml
version: '3'

services:
    frontend:
        build: ./frontend
        image: jcourse:1.0
        ports:
            - 3000:3000
            
    backend:
        build: ./backend
        image: jcourse_api:1.0
        command: gunicorn jcourse.wsgi --workers=5 --bind 0.0.0.0:8000
        ports:
            - 8000:8000
        environment:
            HASH_SALT: 
            SECRET_KEY: 
            POSTGRES_PASSWORD: jcourse
            POSTGRES_HOST: db
            REDIS_HOST: cache
            JACCOUNT_CLIENT_ID: 
            JACCOUNT_CLIENT_SECRET: 
            EMAIL_HOST_USER: 
            EMAIL_HOST_PASSWORD: 
            LOGGING_FILE: ./data/django.log
        volumes:
            - ./static:/django/static
            - ./django-data:/django/data
        depends_on:
            - db
            - cache
        restart: always
        
    db:
        image: postgres:13
        volumes:
            - ./pgdata:/var/lib/postgresql/data
        environment:
            POSTGRES_DB: jcourse
            POSTGRES_USER: jcourse
            POSTGRES_PASSWORD: jcourse
        restart: always
        
    cache:
        image: redis:latest
        restart: always
        
    node-exporter:
        image: prom/node-exporter
        
    prometheus:
        image: prom/prometheus
        volumes:
            - ./prometheus:/etc/prometheus
        ports:
            - 9090:9090
        command:
            - '--config.file=/etc/prometheus/prometheus.yml'
            - '--web.external-url=/prometheus/'
            - '--web.route-prefix=/prometheus/'
        depends_on:
            - backend
            
    grafana:
        image: grafana/grafana
        user: "$UID:$GID"
        environment:
            GF_INSTALL_PLUGINS: "grafana-clock-panel,grafana-simple-json-datasource"
            GF_SERVER_ROOT_URL: "/grafana"
            GF_SERVER_SERVE_FROM_SUB_PATH: "true"
        volumes:
            - ./grafana:/var/lib/grafana
        ports:
            - 3060:3000
        depends_on:
            - prometheus
```

## 3. 各文件夹和相关组件的配置

上述配置文件中涉及到挂载各个文件夹，下面一一列举其作用：

* pgdata：PostgreSQL 的工作目录
* static：Django 的静态资源
* django-data：后端部分功能需要的数据文件夹（对应容器内的 ./data 目录）
* prometheus：prometheus 的工作目录
* grafana：grafana 的工作目录

### Prometheus的配置

prometheus/prometheus.yml 文件

```
global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.
  external_labels:
    monitor: 'codelab-monitor'

scrape_configs:
  - job_name: 'django'
    metrics_path: '/prometheus/metrics'
    static_configs:
      - targets: ['backend:8000']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

## 4. 构建镜像、运行容器

在 jCourse 目录下执行
```
docker-compose up # 加 -d 可以挂在后台
```

后端容器运行之后，收集后端的静态文件到 static 目录。迁移数据库。创建管理员账号
```
docker-compose exec backend python manage.py collectstatic
docker-compose exec backend python manage.py migrate
docker-compose exec backend python manage.py createsuperuser
```

## 5. 配置 NGINX

在 /etc/nginx/site-enabled/ 目录下新建配置文件（比如叫 `course.sjtu.plus`），内容如下：
```nginx
upstream jcourse_api {
    server localhost:8000;
}

upstream jcourse_front {
    server localhost:3000;
}

upstream grafana {
    server localhost:3060;
}

server {
    listen 80;

    server_name course.sjtu.plus;
    
    location = /index.html {
        expires -1;
    }
    
    location @router {
        rewrite ^.*$ /index.html last;
    }

    location /api {
        proxy_pass http://jcourse_api;
        proxy_set_header Host $http_host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    location /oauth {
        proxy_pass http://jcourse_api;
        proxy_set_header Host $http_host;
    }

    location /admin {
        proxy_pass http://jcourse_api;
        proxy_set_header Host $http_host;
    }

    location / {
        proxy_pass http://jcourse_front;
        proxy_set_header Host $http_host;
    }

    location /grafana {
        proxy_pass http://grafana;
        proxy_set_header Host $http_host;
        rewrite ^/grafana/(.*) /$1 break;
    }
    
    location /static {
        expires max;
        autoindex on;
        alias /var/jcourse/static/;
    }

}
```

## 6. 配置HTTPS

1. 使用 Certbot，参考 https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal
2. 使用服务器运营商提供的 SSL 证书，参考对应教程

## 7. 定时备份

设置 crontab 每日 5 点自动从服务器导出数据库内容并上传至七牛云 OSS。

1. 安装七牛云 OSS 命令行工具 https://developer.qiniu.com/kodo/1302/qshell
2. 编写备份脚本 backup.sh 如下
```shell
#!/bin/bash
cd /var/jcourse
docker-compose exec -T backend python3 manage.py clearsessions
docker-compose exec -T backend python3 manage.py dumpdata --exclude=contenttypes -o data/backup-$(date "+%Y%m%d").json.gz
qshell fput jcourse backup-$(date "+%Y%m%d").json.gz ./django-data/backup-$(date "+%Y%m%d").json.gz
```
3. 配置 crontab
命令行输入 `crontab -e` 进入配置文件，输入 `0 5 * * * cd /var/jcourse && bash ./backup.sh > /dev/null 2>&1`
