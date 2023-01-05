---
layout: post
title: Docker容器用戶映射
date: 2015-12-30 15:02:21
author: 任怀林
categories:
- blog
- docker
thumb: docker.png
isCJKLanguage: true
---


root(id = 0) 是容器的默認用戶。Docker image的製作者可以添加新的用戶。    

比如jenkins image的Dockerfile是這樣的：

``` dockerfile
FROM java:8-jdk

RUN apt-get update && apt-get install -y wget git curl zip && rm -rf /var/lib/apt/lists/*

ENV JENKINS_HOME /var/jenkins_home
ENV JENKINS_SLAVE_AGENT_PORT 50000

# Jenkins is run with user `jenkins`, uid = 1000
# If you bind mount a volume from the host or a data container,
# ensure you use the same uid
RUN useradd -d "$JENKINS_HOME" -u 1000 -m -s /bin/bash jenkins

# Jenkins home directory is a volume, so configuration and build history
# can be persisted and survive image upgrades
VOLUME /var/jenkins_home

# `/usr/share/jenkins/ref/` contains all reference configuration we want
# to set on a fresh new installation. Use it to bundle additional plugins
# or config file with your custom jenkins Docker image.
RUN mkdir -p /usr/share/jenkins/ref/init.groovy.d

ENV TINI_SHA 066ad710107dc7ee05d3aa6e4974f01dc98f3888

# Use tini as subreaper in Docker container to adopt zombie processes
RUN curl -fL https://github.com/krallin/tini/releases/download/v0.5.0/tini-static -o /bin/tini && chmod +x /bin/tini \
  && echo "$TINI_SHA /bin/tini" | sha1sum -c -

COPY init.groovy /usr/share/jenkins/ref/init.groovy.d/tcp-slave-agent-port.groovy

ENV JENKINS_VERSION 1.625.3
ENV JENKINS_SHA 537d910f541c25a23499b222ccd37ca25e074a0c


# could use ADD but this one does not check Last-Modified header
# see https://github.com/docker/docker/issues/8331
RUN curl -fL http://mirrors.jenkins-ci.org/war-stable/$JENKINS_VERSION/jenkins.war -o /usr/share/jenkins/jenkins.war \
  && echo "$JENKINS_SHA /usr/share/jenkins/jenkins.war" | sha1sum -c -

ENV JENKINS_UC https://updates.jenkins-ci.org
RUN chown -R jenkins "$JENKINS_HOME" /usr/share/jenkins/ref

# for main web interface:
EXPOSE 8080

# will be used by attached slave agents:
EXPOSE 50000

ENV COPY_REFERENCE_FILE_LOG $JENKINS_HOME/copy_reference_file.log

USER jenkins

COPY jenkins.sh /usr/local/bin/jenkins.sh
ENTRYPOINT ["/bin/tini", "--", "/usr/local/bin/jenkins.sh"]

# from a derived Dockerfile, can use `RUN plugins.sh active.txt` to setup /usr/share/jenkins/ref/plugins from a support bundle
COPY plugins.sh /usr/local/bin/plugins.sh
```

請注意這一行：
```
RUN useradd -d "$JENKINS_HOME" -u 1000 -m -s /bin/bash jenkins
```

製作者創建了一個新用戶叫`jenkins`,指定它的uid爲1000。

如果我們不向容器掛載任何卷，那沒問題。如果我們向容器掛載了卷，就有權限的問題。


假設我們把本地`/opt/jenkins_home`掛載爲容器的`/var/jenkins_home`，而本地的/opt/jenkins_home是屬於root用戶的。

```
# ls /opt
drwxr-xr-x  2 root root  4096 Dec 30 03:11 jenkins_home/
```

那容器在運行時jenkins嘗試向這個目錄寫數據時就會出錯，權限不足。

爲什麼？

因爲容器裏的jenkins用戶的uid是1000,在host主機上，uid爲1000用戶是普通用戶，無法寫root  owned的目錄。

解決方法有兩個。

1 我們把本地的/opt/jenkins_home的所有者修改爲uid爲1000的那個用戶,假設在本地這個用戶是ubuntu。

```
# chown -R ubuntu:ubuntu /opt/jenkins_home
```

這樣我們再把這個目錄掛載到容器裏時，容器裏的jenkins用戶就有寫入的權限了。

或者：

2 運行時通過`-u`指定一個用戶

docker run的 `-u`選項可以指定容器裏的默認用戶，選項的值可以是用戶名，也可以是uid。    
如果是用戶名，這個用戶必須在容器裏是存在的。如果是uid則沒有這個限制。

我們還是用jenkins image來舉例。如果我在運行時指定 `-u ubuntu`，容器就會報錯：
```
ERROR: Cannot start container e0e1201fb3192d0f8e68656de3657e2ae80b111e5f72f56583c908533a89f525: [8] System error: Unable to find user ubuntu
```

回到剛纔我們說的問題，/opt/jenkins_home的owner是root,所以容器無法寫數據。我們可以在運行時使用參數`-u root`,這樣容器內的默認用戶變爲root，就可以向`/opt/jenkins_home`寫入數據了。


總結：image的製作者可以指定容器的默認用戶，因爲作者認爲容器內的進程無需使用root用戶來運行。通常作者指定的用戶的uid都會爲1000（如果image是基於ubuntu或debian的），以保證有本地用戶可以映射。

如果在運行時不想使用作者指定的用戶，可以通過`-u`這個選項來指定一個用戶。選項的值可以是用戶名，也可以是uid。    
如果是用戶名，這個用戶必須在容器裏是存在的。如果是uid則這個用戶在容器裏可以不存在。
