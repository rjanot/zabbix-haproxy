HAProxy Zabbix Discovery and Template
=====================================

[Zabbix](http://zabbix.com) is a powerful open-source monitoring platform, capable of monitoring anything and everything, with the right configuration.
Zabbix's powerful Discovery capability is awesome, making it possible to automatically register hosts as they come online or monitor database servers without having to add individual databases and tables one by one.
This repo contains everything you need to discover and monitor HAProxy frontends, backends and backend servers.

[HAProxy](http://www.haproxy.org/) is an awesome multi-purpose load-balancer.

> HAProxy is a free, very fast and reliable solution offering high availability, load balancing, and proxying for TCP and HTTP-based applications.
> It is particularly suited for very high traffic web sites and powers quite a number of the world's most visited ones. Over the years it has become the de-facto standard opensource load balancer, is now shipped with most mainstream Linux distributions, and is often deployed by default in cloud platforms.


### Prerequisites

* Zabbix Server >= 2.x (tested on 2.2 and 2.4)
* Zabbix Frontend >= 2.x
* HAProxy >= 1.3
* Socat

### Instructions

* Place `userparameter_haproxy.conf` into `/etc/zabbix/zabbix_agentd.d/` directory, assuming you have Include set in `zabbix_agend.conf`, like so:
```
### Option: Include
# You may include individual files or all files in a directory in the configuration file.
# Installing Zabbix will create include directory in /usr/local/etc, unless modified during the compile time.
#
# Mandatory: no
# Default:
Include=/etc/zabbix/zabbix_agentd.d/
```
* Place `haproxy_discovery.sh` into `/usr/local/bin/` directory and make sure it's executable (`sudo chmod +x /usr/local/bin/haproxy_discovery.sh`)
* Import `haproxy_zbx_template.xml` template via Zabbix Web UI interface (provided by `zabbix-frontend-php` package)
* Configure HAProxy control socket
  - [Configure HAProxy](http://cbonte.github.io/haproxy-dconv/configuration-1.5.html#9.2) to listen on `/run/haproxy/info.sock`
  - or set custom socket path in checks (set `{$HAPROXY_SOCK}` template macro to your custom socket path)
  - or update `userparameter_haproxy.conf` and `haproxy_discovery.sh` with your socket path
* Customize your HAProxy config file location via `{$HAPROXY_CONFIG}` template macro, if necessary
```
# haproxy.conf snippet
# haproxy read-only non-admin socket
## (user level permissions are required, admin level will work as well, though not necessary)
global
  stats socket /run/haproxy/info.sock  mode 666 level user

```

>**MAKE SURE TO HAVE APPROPRIATE PERMISSIONS ON HAPROXY SOCKET**  
>You can specify what permissions a stats socker file will be created with in `haproxy.cnf`. When using non-admin socket for stats, it's _mostly_ safe to allow very loose permissions (0666).  
>You can even use something more restrictive like 0660, as long as you add Zabbix Agent's running user (usually "zabbix") to the HAProxy group (usually "haproxy").  
>This way you don't have to prepend `socat` with `sudo` in `userparameter_haproxy.conf` to make sure Zabbix Agent can access the socket. And you don't have to create `/etc/sudoers` entry for Zabbix. And don't need to remember to make it restrictive, avoiding all implication of misconfiguring use of SUDO.  
>The symptom of permissions problem on the socket is the following error from Zabbix Agent:  
>`Received value [] is not suitable for value type [Numeric (unsigned)] and data type [Decimal]`

* Verify on server with HAProxy installed:
```
anapsix@lb1:~$ sudo zabbix_agentd -t haproxy.list.discovery[/run/haproxy/info.sock,FRONTEND]
  haproxy.list.discovery[FRONTEND]              [t|{"data":[{"{#FRONTEND_NAME}":"http-frontend"},{"{#FRONTEND_NAME}":"https-frontend"}]}]
    
anapsix@lb1:~$ sudo zabbix_agentd -t haproxy.list.discovery[/run/haproxy/info.sock,BACKEND]
  haproxy.list.discovery[BACKEND]               [t|{"data":[{"{#BACKEND_NAME}":"www-backend"},{"{#BACKEND_NAME}":"api-backend"}]}]
    
anapsix@lb1:~$ sudo zabbix_agentd -t haproxy.list.discovery[/run/haproxy/info.sock,SERVERS]
  haproxy.list.discovery[SERVERS]               [t|{"data":[{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www01"},{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www02"},{"{#BACKEND_NAME}":"www-backend","{#SERVER_NAME}":"www03"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api01"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api02"},{"{#BACKEND_NAME}":"api-backend","{#SERVER_NAME}":"api03"}]}]
```

* Add hosts with HAProxy installed to just imported Zabbix HAProxy template.
* Wait for discovery.. Frontend(s), Backend(s) and Server(s) should show up under Host Items.  
   An easy way to see all data is via _Overview_ (make sure to pick right Group, one of the "HAProxy" applications and select _Data_ as Type)


### Troubleshooting

#### Discover
```
/usr/local/bin/haproxy_discovery.sh $1 $2
$1 is a path to haproxy socket
$2 is FRONTEND or BACKEND or SERVERS

# /usr/local/bin/haproxy_discovery.sh /var/run/haproxy/info.sock FRONTEND    # second argument is optional
# /usr/local/bin/haproxy_discovery.sh /var/run/haproxy/info.sock BACKEND     # second argument is optional
# /usr/local/bin/haproxy_discovery.sh /var/run/haproxy/info.sock SERVERS     # second argument is optional
```

#### Stats
```
## Bytes In:      echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f9
## Bytes Out:     echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f10
## Session Rate:  echo "show stat" | socat $1 stdio | grep "^$2,$3" | cut -d, -f5
### $1 is a path to haproxy socket
### $2 is a name of the backend, as set in haproxy.cnf
### $3 is a name of the server, as set in haproxy.cnf
# echo "show stat" | socat /var/run/haproxy/info.sock stdio | grep "^www-backend,www01" | cut -d, -f9
# echo "show stat" | socat /var/run/haproxy/info.sock stdio | grep "^www-backend,BACKEND" | cut -d, -f10
# echo "show stat" | socat /var/run/haproxy/info.sock stdio | grep "^https-frontend,FRONTEND" | cut -d, -f5
# echo "show stat" | socat /var/run/haproxy/info.sock stdio | grep "^api-backend,api02" | cut -d, -f18 | cut -d\  -f1
```

#### More
Take a look at the out put of the following to learn more about what is available though HAProxy socket
```
echo "show stat" | socat /var/run/haproxy/info.sock stdio
```

### License

[MIT License](http://opensource.org/licenses/MIT)

    The MIT License (MIT)
    
    Copyright (c) 2015 "Anastas Dancha <anapsix@random.io>"
    
    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    
    The above copyright notice and this permission notice shall be included in
    all copies or substantial portions of the Software.
    
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
    THE SOFTWARE.
