https://kubernetes.default.svc.cluster.local





http://jenkins-master.devops.svc.cluster.local:8080



ccr.ccs.tencentyun.com/tsf_100005250331/cdp-jenkins-slave



/var/run/docker.sock







这篇博客给我给很大的帮助。尤其是50000端口所使用的协议，默认是用加密的，会导致slave连接不上master。

[https://devops.com/kubernetes-jenkins-master-slave-scaling-the-scalability-issue/](https://devops.com/kubernetes-jenkins-master-slave-scaling-the-scalability-issue/)
