- swarm-node-1 -> 172.31.13.115 - 65.0.6.37
- swarm-node-2 -> 172.31.1.231  - 3.111.36.94
- swarm-node-3 -> 172.31.2.80   - 13.127.59.45


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
sudo docker service create --mode replicated --replicas 6 --name minc-Service --network swarm-network --endpoint-mode dnsrr tanmoysrt/minc:v2

sudo docker service create --mode replicated --replicas 6 --name nginx-Service --network swarm-network --endpoint-mode dnsrr nginx

sudo docker service create --mode replicated --replicas 6 --name apache-Service --network swarm-network --endpoint-mode dnsrr httpd
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
    
    default_backend no-match
    
    use_backend stat if { path -i /my-stats }
    
    acl minc.tanmoy.info hdr(host) -i minc.tanmoy.info
    use_backend be_minc_service if minc.tanmoy.info
    
    acl nginx.tanmoy.info hdr(host) -i nginx.tanmoy.info
    use_backend be_nginx_service if nginx.tanmoy.info
    
    acl apache.tanmoy.info hdr(host) -i apache.tanmoy.info
    use_backend be_apache_service if apache.tanmoy.info

backend no-match
  mode http
  http-request deny deny_status 400

backend be_minc_service
    balance roundrobin
    server-template minc- 6 minc-Service:3000 check resolvers docker init-addr libc,none

backend be_nginx_service
    balance roundrobin
    server-template minc- 6 nginx-Service:80 check resolvers docker init-addr libc,none
    
backend be_apache_service
    balance roundrobin
    server-template minc- 6 apache-Service:80 check resolvers docker init-addr libc,none

backend stat
    stats enable
    stats uri /my-stats
    stats refresh 15s
    stats show-legends
    stats show-node
```

Copy this cfg to all node


-> http://minc.tanmoy.info/
-> http://nginx.tanmoy.info/
-> http://apache.tanmoy.info/


## Create haproxy service

#### Deploy in all node and directly bind port to host
```
sudo docker service create \
  --mode global \
  --name haproxy-service \
  --network swarm-network \
  --publish published=80,target=80,protocol=tcp,mode=host \
  --publish published=443,target=443,protocol=tcp,mode=host \
  --mount type=bind,src=/etc/haproxy/,dst=/etc/haproxy/,ro=true \
  --dns=127.0.0.11 \
  haproxytech/haproxy-debian:2.0
```


#### Load balancer -> 52.66.196.121

###### Install 
```
apt-get install haproxy
```
###### Loadbalancer configuration with Offloading
https://www.haproxy.com/blog/haproxy-ssl-termination
```
global
    log /dev/log local0
    log /dev/log local1 notice
    chroot /var/lib/haproxy
    stats socket /run/haproxy/admin.sock mode 660 level admin expose-fd listeners
    stats timeout 30s
    user haproxy
    group haproxy
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5000
    timeout client 50000
    timeout server 50000

frontend main
    bind :80
    bind :443 ssl crt /etc/haproxy/certs/ alpn h2,http/1.1

    redirect scheme https code 301 if !{ ssl_fc }
    
    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    use_backend letsencrypt-backend if letsencrypt-acl
    use_backend stat if { path -i /stats }

    default_backend servers

backend letsencrypt-backend
    server letsencrypt 127.0.0.1:8888

backend servers
    balance roundrobin
    server server1 172.31.13.115:80 check
    server server2 172.31.1.231:80 check
    server server3 172.31.2.80:80 check
    
backend stat
    stats enable
    stats uri /stats
    stats refresh 15s
    stats show-legends
    stats show-node
```
