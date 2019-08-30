---
layout: post
title: 在IBM Bluemix上运行ShadowSocks
date: 2017-10-12 10:38:00
author: 任怀林
categories:
- linux
  thumb: linux.png
---

# 在IBM Bluemix上运行ShadowSocks



## **ShadowSocks Docker Image**

<https://hub.docker.com/r/mritd/shadowsocks/>



```
$ bx login --sso https://api.eu-de.bluemix.net

$ bx cs init

$ bx cs cluster-config mycluster

# 登录到bluemix的registry.
$ bx cr login

# 这个必须有，否则我们无法push镜像
$ bx cr namespace-add harleyren

$ docker tag registry.eu-de.bluemix.net/harleyren/shadowsocks:3.1.0 registry.eu-gb.bluemix.net/harleyren/shadowsocks:3.1.0

$ docker push registry.eu-gb.bluemix.net/harleyren/shadowsocks:3.1.0
```





```
$ kubectl proxy --address="0.0.0.0"
```





```yaml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  # Unique key of the Deployment instance
  name: deployment-shadowsocks
spec:
  # 3 Pods should exist at all times.
  replicas: 1
  template:
    metadata:
      labels:
        # Apply this label to pods and default
        # the Deployment label selector to this value
        app: shadowsocks
    spec:
      containers:
      - name: shadowsocks
        # Run this image
        image: registry.eu-de.bluemix.net/harleyren/shadowsocks:3.1.0
        ports:
          - containerPort: 8388
        env:
          - name: 'SS_MODULE'
            value: 'ss-server'
          - name: 'SS_CONFIG'
            value: '-s 0.0.0.0 -p 8388 -m rc4-md5 -k yousecretkey --fast-open'
         
```





```
$ kubectl expose deployment/deployment-shadowsocks --type=NodePort --port=8388 --name=shadowsocks --target-port=8388

$ kubectl describe service shadowsocks
```

yaml 格式：

```yaml
kind: Service
apiVersion: v1
metadata:
  # Unique key of the Service instance
  name: shadowsocks
spec:
  ports:
  # ShadowSocks Port
  - name: ss
  	# The port that will be exposed by this service.
    port: 8388
    # The port that pod exposed.
    targetPort: 8388
  - name: kcp
  	# The port that will be exposed by this service.
    port: 9000
    # The port that pod exposed.
    targetPort: 9000
    protocol: "UDP"
  selector:
    # Loadbalance traffic across Pods matching
    # this label selector
    app: shadowsocks
  type: NodePort
```