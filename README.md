# kube-ovn-temp


### set --graceful-restart=true
```text
[root@node1 ~]# cat speaker.yaml

kind: DaemonSet
apiVersion: apps/v1
metadata:
  name: kube-ovn-speaker
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: kube-ovn-speaker
  template:
    metadata:
      labels:
        app: kube-ovn-speaker
        component: network
        type: infra
    spec:
      tolerations:
        - operator: Exists
          effect: NoSchedule
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: kube-ovn-speaker
              topologyKey: kubernetes.io/hostname
      priorityClassName: system-cluster-critical
      serviceAccountName: ovn
      hostNetwork: true
      containers:
        - name: ovn-central
          image: "yametech/kube-ovn:v1.7.1"
          imagePullPolicy: IfNotPresent
          command:
            - /kube-ovn/kube-ovn-speaker
          args:
            - --graceful-restart=true
            - --neighbor-address=172.16.128.162
            - --neighbor-as=65002
            - --cluster-as=65000
          env:
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
          resources:
            requests:
              cpu: 500m
              memory: 300Mi
      nodeSelector:
        kubernetes.io/os: "linux"
        ovn.kubernetes.io/bgp: "true"
```

### gobgp setting

```text
[root@node3 ~]# cat gobgpd.yml

global:
    config:
        as: 65002
        router-id: 172.16.128.162
neighbors:
    - config:
        neighbor-address: 172.16.128.160
        peer-as: 65000
      graceful-restart:
        config:
          enabled: true
      afi-safis:
        - config:
            afi-safi-name: ipv4-unicast
          mp-graceful-restart:
            config:
              enabled: true
    - config:
        neighbor-address: 172.16.128.161
        peer-as: 65000
      graceful-restart:
        config:
          enabled: true
      afi-safis:
        - config:
            afi-safi-name: ipv4-unicast
          mp-graceful-restart:
             config:
               enabled: true

```


### start gobgpd

```text

[root@node3 ~]# gobgpd -t yaml -f gobgpd.yml
{"level":"info","msg":"gobgpd started","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Config","level":"info","msg":"Finished reading the config file","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.160 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.160","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.161 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.161","time":"2021-05-08T17:20:56+08:00"}

```

### start ovn-speaker
```text
[root@node1 ~]# k apply -f speaker.yaml
daemonset.apps/kube-ovn-speaker created
[root@node1 ~]# kg po
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-b5648d655-v9jhf                1/1     Running   0          2d2h
coredns-b5648d655-z8nxl                1/1     Running   0          2d2h
etcd-node1                             1/1     Running   0          2d2h
kube-apiserver-node1                   1/1     Running   0          2d2h
kube-controller-manager-node1          1/1     Running   0          2d2h
kube-ovn-cni-5ztc5                     1/1     Running   0          2d2h
kube-ovn-cni-dtzjr                     1/1     Running   0          2d2h
kube-ovn-controller-66d5f44b9b-7flqr   1/1     Running   0          2d2h
kube-ovn-pinger-rfnmm                  1/1     Running   0          2d2h
kube-ovn-pinger-whjdt                  1/1     Running   0          2d2h
kube-ovn-speaker-mnzp2                 1/1     Running   0          3s
kube-ovn-speaker-zw98v                 1/1     Running   0          3s
kube-proxy-4c5xm                       1/1     Running   0          2d2h
kube-proxy-lh9mf                       1/1     Running   0          2d2h
kube-scheduler-node1                   1/1     Running   0          2d2h
nginx-deployment-766f95f789-8spg5      1/1     Running   0          2d2h
nginx-deployment-766f95f789-98vl5      1/1     Running   0          2d2h
ovn-central-5bd4b968bd-k75dw           2/2     Running   0          2d2h
ovs-ovn-dnt4m                          1/1     Running   0          2d2h
ovs-ovn-p4d8f                          1/1     Running   0          2d2h
```

### gobgpd message

```text
[root@node3 ~]# gobgpd -t yaml -f gobgpd.yml
{"level":"info","msg":"gobgpd started","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Config","level":"info","msg":"Finished reading the config file","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.160 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.160","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.161 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.161","time":"2021-05-08T17:20:56+08:00"}



{"Key":"172.16.128.161","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:14+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:15+08:00"}

```

### gobgp status

```text
[root@node3 ~]# gobgp n
Peer              AS  Up/Down State       |#Received  Accepted
172.16.128.160 65000 00:01:53 Establ      |        2         2
172.16.128.161 65000 00:01:54 Establ      |        2         2

[root@node3 ~]# gobgp n 172.16.128.160
BGP neighbor is 172.16.128.160, remote AS 65000
  BGP version 4, remote router ID 172.16.128.160
  BGP state = ESTABLISHED, up for 00:02:54
  BGP OutQ = 0, Flops = 0
  Hold time is 90, keepalive interval is 30 seconds
  Configured hold time is 90, keepalive interval is 30 seconds

  Neighbor capabilities:
    multiprotocol:
        ipv4-unicast:	advertised and received
    route-refresh:	advertised and received
    extended-nexthop:	advertised
        Local:  nlri: ipv4-unicast, nexthop: ipv6
    graceful-restart:	advertised and received
        Local: restart time 90 sec
	    ipv4-unicast
        Remote: restart time 90 sec, restart flag set
	    ipv4-unicast, forward flag set
    4-octet-as:	advertised and received
  Message statistics:
                         Sent       Rcvd
    Opens:                  1          1
    Notifications:          0          0
    Updates:                1          2
    Keepalives:             6          6
    Route Refresh:          0          0
    Discarded:              0          0
    Total:                  8          9
  Route statistics:
    Advertised:             1
    Received:               2
    Accepted:               2

[root@node3 ~]# gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.16.0.0/16         172.16.128.161       65000                00:04:01   [{Origin: i}]
*  10.16.0.0/16         172.16.128.160       65000                00:04:00   [{Origin: i}]
*> 10.16.0.21/32        172.16.128.161       65000                00:04:01   [{Origin: i}]
*  10.16.0.21/32        172.16.128.160       65000                00:04:00   [{Origin: i}]
```

### excute delete ovn-spkear daemonset

```text
[root@node1 ~]# k delete -f speaker.yaml
daemonset.apps "kube-ovn-speaker" deleted

//ovn-speaker already delete
[root@node1 ~]# kg po
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-b5648d655-v9jhf                1/1     Running   0          2d2h
coredns-b5648d655-z8nxl                1/1     Running   0          2d2h
etcd-node1                             1/1     Running   0          2d2h
kube-apiserver-node1                   1/1     Running   0          2d2h
kube-controller-manager-node1          1/1     Running   0          2d2h
kube-ovn-cni-5ztc5                     1/1     Running   0          2d2h
kube-ovn-cni-dtzjr                     1/1     Running   0          2d2h
kube-ovn-controller-66d5f44b9b-7flqr   1/1     Running   0          2d2h
kube-ovn-pinger-rfnmm                  1/1     Running   0          2d2h
kube-ovn-pinger-whjdt                  1/1     Running   0          2d2h
kube-proxy-4c5xm                       1/1     Running   0          2d2h
kube-proxy-lh9mf                       1/1     Running   0          2d2h
kube-scheduler-node1                   1/1     Running   0          2d2h
nginx-deployment-766f95f789-8spg5      1/1     Running   0          2d2h
nginx-deployment-766f95f789-98vl5      1/1     Running   0          2d2h
ovn-central-5bd4b968bd-k75dw           2/2     Running   0          2d2h
ovs-ovn-dnt4m                          1/1     Running   0          2d2h
ovs-ovn-p4d8f                          1/1     Running   0          2d2h


//graceful-restart
[root@node3 ~]# gobgpd -t yaml -f gobgpd.yml
{"level":"info","msg":"gobgpd started","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Config","level":"info","msg":"Finished reading the config file","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.160 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.160","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.161 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.161","time":"2021-05-08T17:20:56+08:00"}



{"Key":"172.16.128.161","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:14+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:15+08:00"}
{"Key":"172.16.128.161","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"peer graceful restart","time":"2021-05-08T17:27:05+08:00"}
{"Key":"172.16.128.161","Reason":"graceful-restart","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"Peer Down","time":"2021-05-08T17:27:05+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"peer graceful restart","time":"2021-05-08T17:27:06+08:00"}
{"Key":"172.16.128.160","Reason":"graceful-restart","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"Peer Down","time":"2021-05-08T17:27:06+08:00"}
```

### show rib

```text
[root@node3 ~]# gobgp global rib
   Network              Next Hop             AS_PATH              Age        Attrs
*> 10.16.0.0/16         172.16.128.161       65000                00:04:01   [{Origin: i}]
*  10.16.0.0/16         172.16.128.160       65000                00:04:00   [{Origin: i}]
*> 10.16.0.21/32        172.16.128.161       65000                00:04:01   [{Origin: i}]
*  10.16.0.21/32        172.16.128.160       65000                00:04:00   [{Origin: i}]
```

### after  90s 
``` text
//graceful restart timer expired
[root@node3 ~]# gobgpd -t yaml -f gobgpd.yml
{"level":"info","msg":"gobgpd started","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Config","level":"info","msg":"Finished reading the config file","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.160 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.160","time":"2021-05-08T17:20:56+08:00"}
{"level":"info","msg":"Peer 172.16.128.161 is added","time":"2021-05-08T17:20:56+08:00"}
{"Topic":"Peer","level":"info","msg":"Add a peer configuration for:172.16.128.161","time":"2021-05-08T17:20:56+08:00"}



{"Key":"172.16.128.161","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:14+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_OPENCONFIRM","Topic":"Peer","level":"info","msg":"Peer Up","time":"2021-05-08T17:22:15+08:00"}
{"Key":"172.16.128.161","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"peer graceful restart","time":"2021-05-08T17:27:05+08:00"}
{"Key":"172.16.128.161","Reason":"graceful-restart","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"Peer Down","time":"2021-05-08T17:27:05+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"peer graceful restart","time":"2021-05-08T17:27:06+08:00"}
{"Key":"172.16.128.160","Reason":"graceful-restart","State":"BGP_FSM_ESTABLISHED","Topic":"Peer","level":"info","msg":"Peer Down","time":"2021-05-08T17:27:06+08:00"}
{"Key":"172.16.128.161","State":"BGP_FSM_ACTIVE","Topic":"Peer","level":"warning","msg":"graceful restart timer expired","time":"2021-05-08T17:28:35+08:00"}
{"Key":"172.16.128.160","State":"BGP_FSM_ACTIVE","Topic":"Peer","level":"warning","msg":"graceful restart timer expired","time":"2021-05-08T17:28:36+08:00"}

// gobgp global rib
[root@node3 ~]# gobgp global rib
Network not in table
```
