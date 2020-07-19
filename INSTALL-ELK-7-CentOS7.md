# Installation instructions for CentOS 7

## Assumptions
server.example.com __(ELK master)__

client.example.com __(client machine)__

## ELK Stack installation on server.example.com
###### Import PGP Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[elasticsearch]
name=Elasticsearch repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=0
autorefresh=1
type=rpm-md

[kibana-7.x]
name=Kibana repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
### Elasticsearch
###### Install Elasticsearch
```
yum install -y --enablerepo=elasticsearch elasticsearch
```
###### Enable and start elasticsearch service
```
systemctl daemon-reload
systemctl enable elasticsearch
systemctl start elasticsearch
```
### Kibana
###### Install kibana
```
yum install -y kibana
```
###### Enable and start kibana service
```
systemctl daemon-reload
systemctl enable kibana
systemctl start kibana
```

### Logstash
###### Install logstash
```
yum install -y logstash
```
###### Optional Step. This step is required to enable secure connection between Logstash server and client machines that sends data to Logstash
###### Generate SSL Certificates
```
openssl req -subj '/CN=server.example.com/' -x509 -days 3650 -nodes -batch -newkey rsa:2048 -keyout /etc/pki/tls/private/logstash.key -out /etc/pki/tls/certs/logstash.crt
```
###### Configure Logstash
###### Create Logstash config file
```
vi /etc/logstash/conf.d/logstash-simple.conf
```
Paste the below content
```
input {
  beats {
    port => 5044
  }
}

filter {
    if [type] == "syslog" {
        grok {
            match => {
                "message" => "%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}"
            }
            add_field => [ "received_at", "%{@timestamp}" ]
            add_field => [ "received_from", "%{host}" ]
        }
        syslog_pri { }
        date {
            match => [ "syslog_timestamp", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
        }
    }
}

output {
    elasticsearch {
        hosts => "localhost:9200"
        index => "%{[@metadata][beat]}-%{+YYYY.MM.dd}"
    }
}
```
###### Enable and Start logstash service
```
systemctl enable logstash
systemctl start logstash
```
## FileBeat installation on client.example.com
###### Import PGP Key
```
rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
###### Create Yum repository
```
cat >>/etc/yum.repos.d/elk.repo<<EOF
[elastic-7.x]
name=Elastic repository for 7.x packages
baseurl=https://artifacts.elastic.co/packages/7.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
EOF
```
###### Install Filebeat
```
yum install -y filebeat
```
###### Copy SSL certificate from server.example.com
###### This step is required only if ssl is enabled on Logstash Server
```
scp server.example.com:/etc/pki/tls/certs/logstash.crt /etc/pki/tls/certs/
```
###### Configure Filebeat 
https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-configuration.html

Edit the config file /etc/filebeat/filebeat.yml
enabled: true

paths:
  - /var/log/messages
  
Comment output.elasticsearch
Comment hosts: ["localhost:9200"]

Uncomment output.logstash
hosts: ["server.example.com:5044"]
###### Optional - If ssl is enabled in Logstash
Uncomment ssl.certificate_authorities: ["/etc/pki/tls/certs/logstash.crt "]

setup.kibana:
host: "server.example.com:5601"

systemctl enable filebeat
systemctl start filebeat

###### Enable and start Filebeat service
```
systemctl enable filebeat
systemctl start filebeat
```
### Configure Kibana Dashboard
All done. Now you can head to Kibana dashboard and add the index.


