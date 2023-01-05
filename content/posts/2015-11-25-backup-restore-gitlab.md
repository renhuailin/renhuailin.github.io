---
layout: post
title: 升級gitlab从7.9.4到8.2.2
date: 2015-11-25 14:16:00
author: 任怀林
categories:
- blog
- gitlab
thumb: gitlab.png
isCJKLanguage: true
---
公司的gitlab一直是运行在ovm的虚拟机里的，版本还是6.7.5。版本有点老了，最近在研究docker，于是想把gitlab迁移到docker container里去。发现真的有人已经做了gitlab的image了，真心赞。

# 備份原來的gitlab
```
$ docker exec -it gitlab sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
```
到gitlab的data/backups目錄，把最近的備份copy出來。
gitlab的备份是类似`1449806583_gitlab_backup.tar`这样的文件名，请注意下划线前面的数字，我们将在恢复的时候用到。


查看當前的版本
查看當前的版本，進入gitlab，在左側的菜單欄裏選`help`,可以看到當前版本。我們要先運行舊版本的container,然後pull新版本的gitlab image回來升級。

OK，看了一下我現在的版本7.9.4


# Pull 各種 images

Next , pull gitlab 7.9.4 image from docker hub. 哈.

```
# docker pull sameersbn/gitlab:7.9.4
```

把最新版本的8.2.2 pull回來。

```
# docker pull sameersbn/gitlab:8.2.2
```

網上說MySQL 5.7是相當重要的一個版本，支持好多令人激動的新特性，那我就5.7吧，用docker好處就是能用最新版本的軟件，：）

```
# docker pull mysql:5.7
```

最後要把redis pull回來，因爲要用到。

```
# docker pull sameersbn/redis:latest
```

然後要把MySQL配置好，請參考[我的上篇文章](http://www.renhl.com/blog/openstack/move-gitlab-to-docker-container/)，這裏就不再贅述了。

# 使用docker-compose啓動舊版本gitlab

爲了方便維護，我使用了docker-compose。這是我的配置文件：

docker-compose.yml

``` yaml

gitlab:
  image: sameersbn/gitlab:7.9.4
  command: app:start
  container_name: gitlab
  ports:
    - "80:80"
    - "8443:443"
  environment:
    - GITLAB_HOST=gitlab.renhl.com
    - DB_USER=gitlab
    - "DB_PASS=dbpass"
    - DB_NAME=gitlabhq_production
    - GITLAB_TIMEZONE=Beijing
    - GITLAB_GRAVATAR_ENABLED=false
    - GITLAB_BACKUPS=daily
    - "NGINX_MAX_UPLOAD_SIZE=100m"
    - "UNICORN_TIMEOUT=120"
  volumes:
    - /opt/data/gitlab:/home/git/data
  links:
    - mysql:mysql
    - redis:redisio

mysql:
  container_name: mysql
  image: mysql:5.7
  environment:
    - MYSQL_ROOT_PASSWORD=mysql
  volumes:
    - /opt/data/mysql:/var/lib/mysql

redis:
#  container_name: redis
  image: sameersbn/redis:latest
```

啓動gitlab container.

```
# docker-compose up -d
```




``` yaml

gitlab:
  image: sameersbn/gitlab:8.2.2
  command: app:start
  container_name: gitlab
  ports:
    - "80:80"
    - "8443:443"
  environment:
    - GITLAB_HOST=gitlab.china-ops.com
    - DB_USER=gitlab
    - "DB_PASS=1q2w3e4r"
    - DB_NAME=gitlabhq_production
    - GITLAB_TIMEZONE=Beijing
    - GITLAB_GRAVATAR_ENABLED=false
    - GITLAB_BACKUPS=daily
    - "NGINX_MAX_UPLOAD_SIZE=100m"
    - "UNICORN_TIMEOUT=120"
    - GITLAB_SECRETS_DB_KEY_BASE=fvXhxg7tthcg4jpxpfg9MbrWJbbHTqsRj3xpLNxdrMpsWmgnMNjRdhc73qX7dsgz
  volumes:
    - /opt/data/gitlab:/home/git/data
  links:
    - mysql:mysql
    - redis:redisio

mysql:
  container_name: mysql
  image: mysql:5.7
  environment:
    - MYSQL_ROOT_PASSWORD=mysql
  volumes:
    - /opt/data/mysql:/var/lib/mysql

redis:
#  container_name: redis
  image: sameersbn/redis:latest
```


恢復備份：

```
docker run --name gitlab --rm  -p 80:80  -p 8443:443   --link mysql:mysql   --link redis:redisio   -e "DB_USER=gitlab" -e "DB_PASS=somepass"   -e "DB_NAME=gitlabhq_production"   -e "GITLAB_HOST=gitlab.china-ops.com"   -e "GITLAB_TIMEZONE=Beijing"   -e "GITLAB_GRAVATAR_ENABLED=false"  -e 'GITLAB_BACKUPS=daily' -e "NGINX_MAX_UPLOAD_SIZE=100m" -e "UNICORN_TIMEOUT=120" -e "GITLAB_SECRETS_DB_KEY_BASE=fvXhxg7tthcg4jpxpfg9MbrWJbbHTqsRj3xpLNxdrMpsWmgnMNjRdhc73qX7dsgz"  -v /opt/data/gitlab:/home/git/data   sameersbn/gitlab:7.9.4  app:rake gitlab:backup:restore BACKUP=1448510466
```

後面的那個數字一定要是你的備份的號。


# pull gitlab 8.2.0 image from docker hub.

docker pull sameersbn/gitlab:8.2.0
docker pull mysql:5.7
docker pull sameersbn/redis:latest

mkdir -p /opt/data/gitlab/data
mkdir -p /opt/data/mysql


$ docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag

``` yaml
mysql:
  container_name: mysql
  image: mysql:5.7
  environment:
    - MYSQL_ROOT_PASSWORD=mysql
  volumes:
    - /opt/data/mysql:/var/lib/mysql
docker
```


``` sql
CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER \
ON `gitlabhq_production`.* TO 'gitlab'@'172.17.%.%' IDENTIFIED BY 'dbpassword';
FLUSH PRIVILEGES;
```

pwgen -Bsv1 64

fvXhxg7tthcg4jpxpfg9MbrWJbbHTqsRj3xpLNxdrMpsWmgnMNjRdhc73qX7dsgz



``` yaml
gitlab:
  image: sameersbn/gitlab:8.2.0
  container_name: gitlab
  ports:
    - "80:80"
    - "8443:443"
  environment:
    - DB_USER=gitlab
    - "DB_PASS=1q2w3e4r"
    - DB_NAME=gitlabhq_production
    - GITLAB_TIMEZONE=Beijing
    - GITLAB_GRAVATAR_ENABLED=false
    - GITLAB_BACKUPS=daily
    - "NGINX_MAX_UPLOAD_SIZE=100m"
    - "UNICORN_TIMEOUT=120"
    - GITLAB_SECRETS_DB_KEY_BASE=fvXhxg7tthcg4jpxpfg9MbrWJbbHTqsRj3xpLNxdrMpsWmgnMNjRdhc73qX7dsgz
  volumes:
    - /opt/data/gitlab:/home/git/data
  links:
    - mysql:mysql
    - redis:redisio
mysql:
  container_name: mysql
  image: mysql:5.7
  environment:
    - MYSQL_ROOT_PASSWORD=mysql
  volumes:
    - /opt/data/mysql:/var/lib/mysql

redis:
  container_name: redis
  image: sameersbn/redis:latest
```
