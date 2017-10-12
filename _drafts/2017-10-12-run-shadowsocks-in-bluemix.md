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
        image: registry.ng.bluemix.net/harleyren/shadowsocks:3.1.0
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
```



`kubectl describe service shadowsocks`







#参考：　