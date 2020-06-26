# NGINX Plus API Dynamic Configuration of Upstreams

## Introduction

This module will explain how to use the NGINX Plus API to dynamicly configure NGINX upstreams groups

With NGINX Plus, configuration of upstream servers in a server group can be modified on-the-fly without reloading the 
NGINX servers and its configuration files. This is useful for:

* **Autoscaling**, when you need to add more servers
* **Maintenance**, when you need to remove a server, specify a backup server, or take a server down temporarily
* **Quick setup**, when you need to change upstream server settings such as server weight, active connections, slow start, 
  failure timeouts.
* **Monitoring**, when you get the state of the server or server group with one command

All these changes are made with the NGINX Plus REST API interface with API commands. By default changes made with the 
API are only stored only in the shared memory zone and so the changes are discarded when the NGINX Plus configuration 
file is reloaded. 

In this Module we will configure upstream servers and upstream server groups dynamically (on-the-fly) with the NGINX 
Plus REST API and use the `state` directive so on-the-fly configurations persistent through NGINX reloads. 



## Task 1: 

1. Inspect the upstream server group called `dynamic` has been defined in `upstreams.conf`

 * The `zone` directive configures a zone in the shared memory and sets the zone name and size. The configuration of the 
   server group is kept in this zone, so all worker processes use the same configuration. In our example, the `zone` is 
   also  named `dynamic` and is 64k megabyte in size.
 * The `state` directive configures Persistence of Dynamic Configuration by writing the state information to a file that 
   persists during a reload. The recommended path for Linux distributions is `/var/lib/nginx/state/`


```nginx

# /etc/nginx/conf.d/upstream.conf 

upstream dynamic {
    # Specify a file that keeps the state of the dynamically configurable group:
    state /var/lib/nginx/state/servers.conf;

    zone dynamic 64k;

    # Including keep alive connections are bonus points
    keepalive 32;
}
```

2. On the NGINX plus instance, confirm that the state file is at a empty state

```bash
cat /var/lib/nginx/state/servers.conf
```

3. On the NGINX dashboard, the upstream, `dynamic`, should be empty:

![nginx plus dashboard showing the empty dynamic upstream](media/2020-06-23_16-26.png)

4. We can also confirm the empty state of our upstream, `dynamic`, using our API

```bash
curl -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers | jq

[]
```

5.  Lets now add a two servers, `web1:80` and `web2:80` to the `dynamic` upstream group using the API

```bash
# Add web1:80:
curl -s -X \
POST http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers \
-H 'Content-Type: text/json; charset=utf-8' \
-d @- <<'EOF'

{
  "server": "web1:80",
  "weight": 1,
  "max_conns": 0,
  "max_fails": 1,
  "fail_timeout": "10s",
  "slow_start": "0s",
  "route": "",
  "backup": false,
  "down": false
}
EOF

# Add web2:80:
curl -s -X \
POST http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers \
-H 'Content-Type: text/json; charset=utf-8' \
-d @- <<'EOF'

{
  "server": "web2:80",
  "weight": 1,
  "max_conns": 0,
  "max_fails": 1,
  "fail_timeout": "10s",
  "slow_start": "0s",
  "route": "",
  "backup": false,
  "down": false
}
EOF
```

1.  Lets now mark a server as down, `web3:80` to the `dynamic` upstream group using the API

```bash
# Add web3:80:
curl -s -X \
POST http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers \
-H 'Content-Type: text/json; charset=utf-8' \
-d @- <<'EOF'

{
  "server": "web3:80",
  "weight": 1,
  "max_conns": 0,
  "max_fails": 1,
  "fail_timeout": "10s",
  "slow_start": "10s",
  "route": "",
  "backup": true,
  "down": true
}
EOF
```

7. Once again list out the servers in our upstream, `dynamic`, and view the changes made

```bash
curl -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers | jq

[
  {
    "id": 0,
    "server": "web1:80",
    "weight": 1,
    "max_conns": 0,
    "max_fails": 1,
    "fail_timeout": "10s",
    "slow_start": "0s",
    "route": "",
    "backup": false,
    "down": false
  },
  {
    "id": 1,
    "server": "web1:80",
    "weight": 1,
    "max_conns": 0,
    "max_fails": 1,
    "fail_timeout": "10s",
    "slow_start": "0s",
    "route": "",
    "backup": false,
    "down": false
  },
  {
    "id": 2,
    "server": "web3:80",
    "weight": 1,
    "max_conns": 0,
    "max_fails": 1,
    "fail_timeout": "10s",
    "slow_start": "10s",
    "route": "",
    "backup": false,
    "down": true
  }
]

```

8. We can also confirm that the state file has been updated:

```bash
cat /var/lib/nginx/state/servers.conf

server web1:80 resolve;
server web1:80 resolve;
server web3:80 resolve down;
```

9. It is possible to also remove a server from the upstream group:

```bash
curl -X DELETE -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers/0 | jq

[
  {
    "id": 1,
    "server": "web1:80",
    "weight": 1,
    "max_conns": 0,
    "max_fails": 1,
    "fail_timeout": "10s",
    "slow_start": "0s",
    "route": "",
    "backup": false,
    "down": false
  },
  {
    "id": 2,
    "server": "web3:80",
    "weight": 1,
    "max_conns": 0,
    "max_fails": 1,
    "fail_timeout": "10s",
    "slow_start": "10s",
    "route": "",
    "backup": false,
    "down": true
  }
]

```

10. To add our down server back to rotation and accept live traffic we need to set `down: true` (Note the correct ID):

```bash
# Find the ID of the backup server to '"backup": false', i.e. live
curl -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers | jq '.[]  | select(.down==true)'

{
  "id": 2,
  "server": "web3:80",
  "weight": 1,
  "max_conns": 0,
  "max_fails": 1,
  "fail_timeout": "10s",
  "slow_start": "0s",
  "route": "",
  "backup": false,
  "down": true
}

```

```bash
# Set server to '"backup": false', i.e. live
curl -X PATCH -d '{ "down": false }' -s 'http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers/2'
```

11.  Once again, list out servers in our upstream, `dynamic`

```bash
curl -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers | jq
```

12. Lets reload NGINX and check that changes made using the API has persisted

```bash
# inspect the state of out state file:
cat /var/lib/nginx/state/servers.conf

server web1:80 resolve;
server web3:80 resolve;

# Reload NGINX
nginx -s reload

# Lastly, list out servers in our upstream, `dynamic`
curl -s http://nginx-plus-1:8080/api/6/http/upstreams/dynamic/servers | jq
```