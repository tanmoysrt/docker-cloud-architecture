swarm-node-1 -> 172.31.42.111 - 65.0.29.93
swarm-node-2 -> 172.31.43.230 - 43.205.118.91
swarm-node-3 -> 172.31.38.157 - 65.0.76.216


## Docker install
```
sudo apt update -y
sudo apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io
```

## Create Overlay Network
```
sudo docker network create --attachable --driver overlay swarm-network
```

## Create Service
```
sudo docker service create --mode replicated --replicas 0 --name minc-Service --network swarm-network --endpoint-mode dnsrr tanmoysrt/minc:v2
```


## haproxy.cfg
```
global
    log          fd@2 local2
    chroot       /var/lib/haproxy
    pidfile      /var/run/haproxy.pid
    maxconn      4000
    user         haproxy
    group        haproxy
    stats socket /var/lib/haproxy/stats expose-fd listeners
    master-worker

resolvers docker
    nameserver dns1 127.0.0.11:53
    resolve_retries 3
    timeout resolve 1s
    timeout retry   1s
    hold other      10s
    hold refused    10s
    hold nx         10s
    hold timeout    10s
    hold valid      10s
    hold obsolete   10s

defaults
    timeout connect 10s
    timeout client 30s
    timeout server 30s
    log global
    mode http
    option httplog

frontend  fe_web
    bind *:80
    use_backend stat if { path -i /my-stats }
    default_backend be_minc_service 

backend be_minc_service
    balance roundrobin
    server-template minc- 12 minc-Service:3000 check resolvers docker init-addr libc,none

backend be_minc_service_wrong_case
    balance roundrobin
    server-template minc- 12 minc-service:3000 check resolvers docker init-addr libc,none

backend stat
    stats enable
    stats uri /my-stats
    stats refresh 15s
    stats show-legends
    stats show-node
```

Copy this cfg to all node

## Create haproxy service

#### With Replications

```
sudo docker service create \
  --mode replicated \
  --replicas 1 \
  --name haproxy-service \
  --network swarm-network \
  --publish published=80,target=80,protocol=tcp,mode=ingress \
  --publish published=443,target=443,protocol=tcp,mode=ingress \
  --mount type=bind,src=/etc/haproxy/,dst=/etc/haproxy/,ro=true \
  --dns=127.0.0.11 \
  haproxytech/haproxy-debian:2.0
```

#### Make it global
```
sudo docker service create \
  --mode global \
  --name haproxy-service \
  --network swarm-network \
  --publish published=80,target=80,protocol=tcp,mode=ingress \
  --publish published=443,target=443,protocol=tcp,mode=ingress \
  --mount type=bind,src=/etc/haproxy/,dst=/etc/haproxy/,ro=true \
  --dns=127.0.0.11 \
  haproxytech/haproxy-debian:2.0
```
