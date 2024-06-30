## Grafana Loki:
Grafana Loki is a highly efficient log aggregation system designed to be simple, scalable, and cost-effective. Unlike traditional logging systems, Loki is optimized to handle massive amounts of logs without requiring extensive storage or complex configuration.

- Grafana: Grafana is an open-source interactive data-visualization platform.
- Loki: Log aggregation system inspired by Prometheus.
- Promtail: Promtail is an agent which ships the contents of local logs to a private Loki.


```
|---------------|
| Node-1        |
| Promtail      |    
|               |       |----------------|       |------------|
|---------------|       |  Loki          |       |  Grafana   |
                        | 192.168.10.190 |       |------------|
                        |                |        
|---------------|       |                |       |------------|
| Node-2        |       |----------------|       | AlertMGR   |
| Promtail      |                                |------------|
|               |
|---------------|
```



### Installing Loki:

Download Loki binary packages on Server:

```
curl -O -L "https://github.com/grafana/loki/releases/download/v2.9.1/loki-linux-amd64.zip"

unzip "loki-linux-amd64.zip"

chmod a+x "loki-linux-amd64"
```


### Configuration Loki:

Download Loki Config file: 

```
wget https://raw.githubusercontent.com/grafana/loki/main/cmd/loki/loki-local-config.yaml

cp loki-local-config.yaml loki-local-config.yaml.bak
```


```
vim loki-local-config.yaml

auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100


schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper   # or 'tsdb'
      object_store: filesystem
      schema: v11   #v13
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093   #Alertmanager Instance


# By default, Loki will send anonymous, but uniquely-identifiable usage and configuration
# analytics to Grafana Labs. These statistics are sent to https://stats.grafana.org/
#
# Statistics help us better understand how Loki is used, and they show us performance
# levels for most users. This helps us prioritize features and documentation.
# For more information on what's sent, look at
# https://github.com/grafana/loki/blob/main/pkg/analytics/stats.go
# Refer to the buildReport method to see what goes into a report.
#


# If you would like to disable reporting, uncomment the following lines:
#analytics:
#  reporting_enabled: false
```



_For `S3` as a storage:_
```
common:
  path_prefix: /tmp/loki
  storage:
    s3: # The "filesystem" section changes to "s3"
      
      s3: https://storage.yandexcloud.net
      bucketnames: loki-logs
      region: ru-central1
      access_key_id:
      secret_access_key:
  
  replication_factor: 1
```



```
./loki-linux-amd64 --version
```


### Run or Start Loki:

_Run on Screen:_
```
screen -S loki

./loki-linux-amd64 -config.file=loki-local-config.yaml
```


```
netstat -tulpn | grep 3100

tcp6       0      0 :::3100       :::*        LISTEN      10804/./loki-linux-
```


_Run Loki on Systemd:_ [Worked]

```
ll /opt/Loki/loki-linux-amd64
    -rwxr-xr-x 1 root root 66859008 Sep 14  2023 /opt/Loki/loki-linux-amd64


ll /opt/Loki/loki-local-config.yaml
    -rw-r--r-- 1 root root 1451 Jun 30 13:09 /opt/Loki/loki-local-config.yaml
```


```
useradd --system loki
```


```
vim /etc/systemd/system/loki.service

[Unit]
Description=Loki service
After=network.target

[Service]
Type=simple
#User=loki
User=root
ExecStart=/opt/Loki/loki-linux-amd64 -config.file /opt/Loki/loki-local-config.yaml

[Install]
WantedBy=multi-user.target
```


```
systemctl daemon-reload

systemctl start loki
systemctl status loki
systemctl enable loki
```


```
netstat -tulpn | grep 3100

tcp6       0      0 :::3100        :::*      LISTEN      16234/loki-linux-am
```



### Loki Console:

```
http://your_server_ip:3100/metrics
```




---
---



## Promtail installation for Loki Node/client:


### Installing Promtail:
Promtail binary Packages download:

```
wget https://github.com/grafana/loki/releases/download/v2.9.1/promtail-linux-amd64.zip

unzip promtail-linux-amd64.zip

chmod +x promtail-linux-amd64
```


### Configuration Promtail:
Download Promtail Config file: 

```
wget https://raw.githubusercontent.com/grafana/loki/main/clients/cmd/promtail/promtail-local-config.yaml
```



_For ssh Log:_
```
vim promtail-local-config.yaml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

#Loki Server IP:
clients:
  - url: http://192.168.10.190:3100/loki/api/v1/push

scrape_configs:
- job_name: system_node1
  static_configs:
  - targets:
      - localhost  #Promtail target is localhost
    labels:
      job: sshlogs
      #job: varlogs
      #__path__: /var/log/*log
      #__path__: /var/log/auth.log
      __path__: /var/log/secure

```



_For HTTPD/Apache2 Log:_
```
vim  promtail-local-config.yaml


server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

#Loki Server IP:
clients:
  - url: http://192.168.10.190:3100/loki/api/v1/push

scrape_configs:
- job_name: system_node1
  static_configs:
  - targets:
      - localhost
    labels:
      job: sshlogs
      #job: varlogs
      #__path__: /var/log/*log
      #__path__: /var/log/auth.log
      __path__: /var/log/secure


- job_name: apache2_node1
  static_configs:
  - targets:
      - localhost
    labels:
      job: apachelogs
      __path__: /var/log/apache2/access.log
```



```
./promtail-linux-amd64 --version
```


### Promtail Start:

_Run on Screen:_
```
screen -S promtail

./promtail-linux-amd64 -config.file=promtail-local-config.yaml
```


```
netstat -tulpn | grep 9080

tcp6       0      0 :::9080      :::*         LISTEN      21864/./promtail-li
```


_Run Promtail  on Systemd:_ [Worked]

```
ll /opt/promtail/promtail-local-config.yaml
    -rw-r--r-- 1 root root 384 Jun 30 13:38 /opt/promtail/promtail-local-config.yaml

ll /opt/promtail/promtail-linux-amd64
    -rwxr-xr-x 1 root root 93775016 Sep 14  2023 /opt/promtail/promtail-linux-amd64
```


```
useradd --system promtail
```


```
vim /etc/systemd/system/promtail.service

[Unit]
Description=Promtail service
After=network.target

[Service]
Type=simple
#User=promtail
User=root
ExecStart=/opt/promtail/promtail-linux-amd64 -config.file /opt/promtail/promtail-local-config.yaml

[Install]
WantedBy=multi-user.target
```


```
systemctl daemon-reload

systemctl start promtail
systemctl status promtail
systemctl enable promtail
```


```
netstat -tulpn | grep 9080
```


```
cat /tmp/positions.yaml

positions:
  /var/log/secure: "5765"
```


```
curl -v http://localhost:3100/loki/api/v1/push
```


### Promtail Console:
```
http://your_server_ip:9080/targets
```






---
---



## Grafana Install:
Installing and running Grafana using the binary method involves downloading the Grafana binary package, extracting it, and configuring it to run on your system. 


### Download the Grafana Binary:
```
wget https://dl.grafana.com/oss/release/grafana-9.5.2.linux-amd64.tar.gz

tar -zxvf grafana-9.5.2.linux-amd64.tar.gz
```


```
cd grafana-9.5.2

ll

drwxr-xr-x  2 root root  4096 May  8  2023 bin
drwxr-xr-x  3 root root  4096 May  8  2023 conf
-rw-r--r--  1 root root 34523 May  8  2023 LICENSE
-rw-r--r--  1 root root   105 May  8  2023 NOTICE.md
drwxr-xr-x  3 root root  4096 May  8  2023 plugins-bundled
drwxr-xr-x 17 root root  4096 May  8  2023 public
-rw-r--r--  1 root root  3082 May  8  2023 README.md
-rw-r--r--  1 root root     5 May  8  2023 VERSION
```


```
ll conf/

-rw-r--r-- 1 root root 57828 May  8  2023 defaults.ini
-rw-r--r-- 1 root root  1045 May  8  2023 ldap_multiple.toml
-rw-r--r-- 1 root root  2986 May  8  2023 ldap.toml
drwxr-xr-x 8 root root  4096 May  8  2023 provisioning
-rw-r--r-- 1 root root 55240 May  8  2023 sample.ini
```


```
ll bin/

-rwxr-xr-x 1 root root 110472416 May  8  2023 grafana
-rwxr-xr-x 1 root root   1483286 May  8  2023 grafana-cli
-rw-r--r-- 1 root root        33 May  8  2023 grafana-cli.md5
-rw-r--r-- 1 root root        33 May  8  2023 grafana.md5
-rwxr-xr-x 1 root root   1483286 May  8  2023 grafana-server
-rw-r--r-- 1 root root        33 May  8  2023 grafana-server.md5
```


```
./bin/grafana-server -v

Version 9.5.2 (commit: cfcea75916, branch: HEAD)
```


### Start Grafana

_Run on Screen:_
```
screen -S grafana

./bin/grafana-server
```

or,

```
nohup ./bin/grafana-server &
```


```
netstat -tlpn | grep 3000

tcp6       0      0 :::3000        :::*      LISTEN      21429/grafana
```


### Grafana Console:

Login using the default credentials: admin/admin

```
http://your_server_ip:3000
```



### Configure Grafana to use Loki as a Data Source:

- Querying Logs:

1. Goto Grafana server: (http://192.168.10.190:3000)
- Home -> Connections -> Connect data -> Select Data sources -> Loki -> click Create a loki data source:
	- Name: Loki-01
	- URL: http://192.168.10.190:3100/    [-> Loki Server IP]
	- click "Save & test"  [-> Output: Data source successfully connected]


2. Verify: 
- Home -> Explore -> 
	- Select: Loki-01
	- Label filters:
		- filename: /var/log/auth.log

    - or,

		- job: sshlogs

	- click: `Run query` top of the right side.



Now, you should have Grafana, Loki, and Promtail set up on your CentOS server. You can start creating dashboards and visualizing logs in Grafana.




---
---




## Loki for Docker:

```
docker network ls

NETWORK ID     NAME         DRIVER    SCOPE
5c918bf885f8   monitoring   bridge    local
```


```
docker pull grafana/loki
docker pull grafana/promtail
docker pull grafana/grafana
```

```
mkdir -p loki/conf/
mkdir -p loki/loki_data/

mkdir -p promtail/conf/
mkdir -p promtail/promtail_data/

mkdir -p grafana/conf/
mkdir -p grafana/grafana_data/
```



```
vim loki/conf/local-config.yaml

auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100


schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093

#analytics:
#  reporting_enabled: false
```


```
vim promtail/conf/config.yml

server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push
  #- url: http://192.168.10.191:3100/loki/api/v1/push

scrape_configs:
- job_name: secure_log_node-191
  static_configs:
  - targets:
      - localhost
    labels:
      job: secure_log_node-191
      __path__: /var/log/secure
```



_Loki:_
```
docker run --name loki -dit -p 3100:3100 --network monitoring --user "$(id -u)" -v "./loki/conf/local-config.yaml:/etc/loki/local-config.yaml" grafana/loki
```


_Promtail:_
```
docker run --name promtail -dit -p 9080:9080 --network monitoring --user "$(id -u)" -v "./promtail/conf/config.yml:/etc/promtail/config.yml" -v "/var/log/secure:/var/log/secure" grafana/promtail
```


_Grafana:_
```
docker run --name grafana -dit -p 3000:3000 --network monitoring --user "$(id -u)" -e "GF_SECURITY_ADMIN_PASSWORD=admin123" grafana/grafana
```


```
docker exec -it promtail bash
cd /etc/promtail/

apt update -y
apt install telnet -y
apt install net-tools iputils-ping -y 

telnet localhost 9080
telnet loki 3100
exit
```



### Docker-compose file: [Worked]


```
vim .env 

#Grafana Admin user pass:
GF_SECURITY_ADMIN_PASSWORD=admin123
```



```
vim docker-compose.yml

version: "3"
services:
  loki:
    image: grafana/loki
    container_name: loki
    ports:
      - "3100:3100"
    user: '0'
    volumes:
      - ./loki/conf:/etc/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    user: '0'
    volumes:
      - ./promtail/conf:/etc/promtail
      #- /var/log:/var/log
      - /var/log/secure:/var/log/secure
    command: -config.file=/etc/promtail/config.yml
    networks:
      - monitoring

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    networks:
      - monitoring
    ports:
      - '3000:3000'
    user: '0'
    volumes:
      #- grafana_data:/var/lib/grafana
      - ./grafana/grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD}

networks:
  monitoring:

volumes:
  grafana_data:
    external: true
```



```
docker-compose -f docker-compose.yml config
docker-compose up -d
```


```
docker-compose ps
  Name                Command               State           Ports
--------------------------------------------------------------------------
grafana    /run.sh                          Up      0.0.0.0:3000->3000/tcp
loki       /usr/bin/loki -config.file ...   Up      0.0.0.0:3100->3100/tcp
promtail   /usr/bin/promtail -config. ...   Up      0.0.0.0:9080->9080/tcp
```




### Configure Grafana to use Loki as a Data Source:

- Querying Logs:

1. Goto Grafana server: (http://192.168.10.190:3000)
- Home -> Connections -> Connect data -> Select Data sources -> Loki -> click Create a loki data source:
	- Name: Loki-01
	- URL: http://192.168.10.190:3100/    [-> Loki Server IP]
	- click "Save & test"  [-> Output: Data source successfully connected]


2. Test: 
- Home -> Explore -> 
	- Select: Loki-01
	- Label filters:
		- filename: /var/log/auth.log 
		- job: sshlogs

	- click: `Run query` top of the right side.





---
---





## Logging With Docker, Promtail and Grafana Loki:

```
```



### Links:
- [Grafana Standalone Binaries Install](https://grafana.com/grafana/download?edition=oss)
- [Install Loki manually](https://grafana.com/docs/loki/latest/setup/install/local/)
- [Loki configuration](https://grafana.com/docs/loki/latest/configure/)
- [Loki with Docker](https://grafana.com/docs/loki/latest/setup/install/docker/)
- [Loki run as a service](https://sbcode.net/grafana/install-loki-service/)
- [Promtail as a Service](https://sbcode.net/grafana/install-promtail-service/)
- [Loki github](https://github.com/grafana/loki/releases/)
- [Visualizing Log Data](https://aws.plainenglish.io/visualizing-log-data-with-grafana-loki-and-promtail-f82179ed1215/)
- [Loki to monitor the logs](https://cylab.be/blog/241/use-loki-to-monitor-the-logs-of-your-docker-compose-application/)
- [Install Grafana Binary](https://www.devopsschool.com/blog/install-and-configure-grafana-in-rhel-7/)

