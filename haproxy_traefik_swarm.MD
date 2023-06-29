swarm-node-1 -> 172.31.13.115 - 65.0.6.37
swarm-node-2 -> 172.31.1.231  - 3.111.36.94
swarm-node-3 -> 172.31.2.80   - 13.127.59.45


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

## Create dockerproxy from main swarm node
```
docker container run \
   -d --privileged \
   --name dockerproxy \
   -v /var/run/docker.sock:/var/run/docker.sock \
   -p 172.31.13.115:2375:2375 \
   --env-file ./dockerproxy.env \
   tecnativa/docker-socket-proxy
```


## Create Service
```
sudo docker service create \
  --mode replicated \
  --replicas 6 \
  --name minc-Service \
  --label "traefik.enable=true" \
  --label "traefik.http.services.minc-Service.loadbalancer.server.port=3000" \
  --label "traefik.http.routers.minc-Service.rule=Host(\`minc.tanmoy.info\`)" \
  --network swarm-network \
  --endpoint-mode dnsrr \
  tanmoysrt/minc:v2

sudo docker service create \
  --mode replicated \
  --replicas 6 \
  --name nginx-Service \
  --label "traefik.enable=true" \
  --label "traefik.http.services.nginx-Service.loadbalancer.server.port=80" \
  --label "traefik.http.routers.nginx-Service.rule=Host(\`nginx.tanmoy.info\`)" \
  --network swarm-network \
  --endpoint-mode dnsrr \
  nginx
  
sudo docker service create \
  --mode replicated \
  --replicas 6 \
  --name apache-Service \
  --label "traefik.enable=true" \
  --label "traefik.http.services.apache-Service.loadbalancer.server.port=80" \
  --label "traefik.http.routers.apache-Service.rule=Host(\`apache.tanmoy.info\`)" \
  --network swarm-network \
  --endpoint-mode dnsrr \
  httpd
  
```

-> http://minc.tanmoy.info/
-> http://nginx.tanmoy.info/
-> http://apache.tanmoy.info/


## Create traefik service

#### Deploy in all node and directly bind port to host
```
docker service create \
  --name traefik-service \
  --mode global \
  --network swarm-network \
  --publish published=80,target=80,protocol=tcp,mode=host \
  --publish published=8080,target=8080,protocol=tcp,mode=host \
  traefik:v2.0 \
  --api.insecure=true \
  --entrypoints.web.address=:80 \
  --providers.docker.endpoint=tcp://172.31.13.115:2375 \
  --providers.docker.swarmMode=true

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

References - 
https://doc.traefik.io/traefik/routing/providers/docker/#port
https://doc.traefik.io/traefik/routing/providers/docker/#port
