---
layout: post
title: 升級gitlab到8.2.0
date: 2015-11-25 14:16:00
author: 任怀林
categories:
- blog
- docker
thumb: openstack.png
---
公司的gitlab一直是运行在ovm的虚拟机里的，版本还是6.7.5。版本有点老了，最近在研究docker，于是想把gitlab迁移到docker container里去。发现真的有人已经做了gitlab的image了，真心赞。




# 備份原來的gitlab
```
$ docker exec -it gitlab sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
```
到gitlab的data/backups目錄，把最近的備份copy出來。


查看當前的版本，進入gitlab，在左側的菜單欄裏選help,可以看到當前版本。

我現在的版本時7.9.4


# pull gitlab 7.9.4 image from docker hub.
docker pull sameersbn/gitlab:7.9.4

## TEST 2

### test3

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


# 1 规划

规划：     
一个容器运行gitlab    
一个容器运行MySQL,然后 link到gitlab上。   
一个容器运行Redis,然后 link到gitlab上。   






# 2 安装gitlab

我们先运行MySQL，

```
$ sudo docker pull sameersbn/mysql:latest
```

在host主机上创建mysql的数据目录。   

```
$ sudo mkdir -p /opt/mysql/data
```

启动MySQL容器。

```
$ sudo docker run --name mysql -d \
    -v /opt/mysql/data:/var/lib/mysql \
    sameersbn/mysql:latest
```


连接到MySQL上，修改授权信息   

``` sh
$ sudo docker exec -it mysql bash
```

创建数据库并授权。    

``` sql
CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER \
ON `gitlabhq_production`.* TO 'gitlab'@'172.17.%.%' IDENTIFIED BY 'dbpassword';
FLUSH PRIVILEGES;
```

# 3 配置redis

Pull image     

``` sh
$ sudo docker pull sameersbn/redis:latest
```

run redis     
``` sh
$ sudo docker run --name=redis -d sameersbn/redis:latest
```

# 4 Gitlab

Pull 先把老版的imagepull回来     

``` sh
$ sudo docker pull sameersbn/gitlab:6.7.5
```

创建数据目录     

``` sh
$ sudo mkdir -p /opt/gitlab/data
```
这个目录会映像到窗口的`/home/git/data`目录上，所以这里保存了所有的数据，请一定不要删除这里的内容。


运行gitlab容器,进行设置，容器会进行数据库的migration等操作。      

``` sh
$ sudo docker run --name gitlab -i -t --rm --link mysql:mysql \
  -e "DB_USER=gitlab" -e "DB_PASS=dbpassword" \
  -e "DB_NAME=gitlabhq_production" \
  -v /opt/gitlab/data:/home/git/data \
  sameersbn/gitlab:6.7.5 app:rake gitlab:setup
```

运行gitlab容器    
{% highlight sh %}
$ sudo docker run --name gitlab -d -P --link mysql:mysql \
  -e "DB_USER=gitlab" -e "DB_PASS=dbpassword" \
  -e "DB_NAME=gitlabhq_production" \
  -v /opt/gitlab/data:/home/git/data \
  sameersbn/gitlab:6.7.5
{% endhighlight %}


{% highlight sh %}
# 从原来gitlab里导出备份
$ cd /home/git/gitlab
$ sudo -u git -H bundle exec rake gitlab:backup:create RAILS_ENV=production
{% endhighlight %}

导出的文件放在`/home/git/gitlab/tmp/backups`这个目录下。

把这个文件 scp到 docker gitlab那台机器的`/opt/gitlab/data/backups`

登录到gitlab的container
{% highlight sh %}
$ sudo docker exec -it gitlab bash
{% endhighlight %}

在容器里执行下面的命令  
{% highlight sh %}
$ cd /home/git/gitlab
$ sudo -u git -H  bundle exec rake gitlab:backup:restore RAILS_ENV=production
$ exit
{% endhighlight %}
这个样数据就全部到新的gitlab上了。
你会发现跟原来的一样。

下面我们来升级gitlab到新版。
{% highlight sh %}
$ sudo docker stop gitlab
$ sudo docker rm gitlab

$ sudo docker run --name gitlab -d -P --link mysql:mysql \
  --link redis:redisio \
  -e "DB_USER=gitlab" -e "DB_PASS=dbpassword" \
  -e "DB_NAME=gitlabhq_production" \
  -v /opt/gitlab/data:/home/git/data \
  sameersbn/gitlab:latest
{% endhighlight %}


配置好以后，把它保存成一个镜像。

{% highlight sh %}
$ sudo docker commit -m "update gitlab.yml ,change host,set timezone to BeiJing" -a "china-ops gitlab v7.9.4" 6af1d0739ae0 china-ops/gitlab:7.9.4
{% endhighlight %}

我原来的想法是修改config/gitlab.yml，把host，timezone等修改好，然后存成一个新的image。
后来发现修改config/gitlab.yml是不生效的，重启container后就会恢复默认值。后来看了文档才知道，
hostname等是通过环境变量来控制的。

用这个镜像来启动一个container
{% highlight sh %}
$ sudo docker run --name gitlab -d  \
  -p 80:80  -p 8443:443 \
  --link mysql:mysql \
  --link redis:redisio \
  -e "DB_USER=gitlab" -e "DB_PASS=dbpassword" \
  -e "DB_NAME=gitlabhq_production" \
  -e "GITLAB_HOST=gitlab.china-ops.com" \
  -e "GITLAB_TIMEZONE=Beijing" \
  -e 'GITLAB_BACKUPS=daily' \
  -e "GITLAB_GRAVATAR_ENABLED=false" \
  -v /opt/gitlab/data:/home/git/data \
  china-ops/gitlab:7.9.4
{% endhighlight %}

参数`-e 'GITLAB_BACKUPS=daily'` 是备份策略，我们设置为每天


默认的密码：    
username: admin@local.host    
password: 5iveL!fe    

参考：　  
[https://github.com/openstack/keystone/blob/master/tools/sample_data.sh](https://github.com/openstack/keystone/blob/master/tools/sample_data.sh)      
[https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst](https://github.com/gc3-uzh-ch/gridka-school/blob/master/tutorial/nova_api.rst)    
[Docker FAQ —— Docker 使用常见问题（持续更新中）](http://dockerpool.com/article/1413082493)
