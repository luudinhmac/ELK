# ELK
# 1. Installing and Configuring Elasticsearch
```
sudo rpm --import https://artifacts.elastic.co/GPG-KEY-elasticsearch
```
Thêm repo  với nội dung dưới đây vào file
```
sudo vi /etc/yum.repos.d/elasticsearch.repo
```
```
[elasticsearch-6.x]
name=Elasticsearch repository for 6.x packages
baseurl=https://artifacts.elastic.co/packages/6.x/yum
gpgcheck=1
gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
enabled=1
autorefresh=1
type=rpm-md
```
**Cài đăt Elasticsearch**
```
sudo yum install elasticsearch
```
**Tìm và sửa lại dòng sau trong file  /etc/elasticsearch/elasticsearch.yml**
```
network.host: localhost
```
Lưu lại và khởi động lại elasticsearch
```
sudo systemctl start elasticsearch
sudo systemctl enable elasticsearch
```
**Test lại elasticsearch service chạy
```
curl -X GET "localhost:9200"
```
Kết quả sẽ hiện như sau
```
Output
{
  "name" : "8oSCBFJ",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "1Nf9ZymBQaOWKpMRBfisog",
  "version" : {
    "number" : "6.5.2",
    "build_flavor" : "default",
    "build_type" : "rpm",
    "build_hash" : "9434bed",
    "build_date" : "2018-11-29T23:58:20.891072Z",
    "build_snapshot" : false,
    "lucene_version" : "7.5.0",
    "minimum_wire_compatibility_version" : "5.6.0",
    "minimum_index_compatibility_version" : "5.0.0"
  },
  "tagline" : "You Know, for Search"
}
```
# 2. Cài đặt và cấu hình Kibana Dashboard
```
sudo yum install kibana
sudo systemctl enable kibana
sudo systemctl start kibana
```
Tạo user và password cho kibana trong file htpassword
```
echo "kibanaadmin:`openssl passwd -apr1`" | sudo tee -a /etc/nginx/htpasswd.users
```
sudo vi /etc/nginx/conf.d/elk.conf
```
server {
    listen 80;

    server_name example.com;

    auth_basic "Restricted Access";
    auth_basic_user_file /etc/nginx/htpasswd.users;

    location / {
        proxy_pass http://localhost:5601;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
```
sudo nginx -t
sudo systemctl restart nginx

sudo setsebool httpd_can_network_connect 1 -P

```

```
http://your_server_ip/status
```
#  3. Cài đặt và cấu hình logstash
```
sudo yum install logstash
```
Tạo 1 file cấu hình để thiết lập Filebeat input
```
sudo vi /etc/logstash/conf.d/02-beats-input.conf
```
```
input {
  beats {
    port => 5044
  }
}
```

```
sudo vi /etc/logstash/conf.d/10-syslog-filter.conf
```
```
filter {
  if [fileset][module] == "system" {
    if [fileset][name] == "auth" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} %{DATA:[system][auth][ssh][method]} for (invalid user )?%{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]} port %{NUMBER:[system][auth][ssh][port]} ssh2(: %{GREEDYDATA:[system][auth][ssh][signature]})?",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: %{DATA:[system][auth][ssh][event]} user %{DATA:[system][auth][user]} from %{IPORHOST:[system][auth][ssh][ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sshd(?:\[%{POSINT:[system][auth][pid]}\])?: Did not receive identification string from %{IPORHOST:[system][auth][ssh][dropped_ip]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} sudo(?:\[%{POSINT:[system][auth][pid]}\])?: \s*%{DATA:[system][auth][user]} :( %{DATA:[system][auth][sudo][error]} ;)? TTY=%{DATA:[system][auth][sudo][tty]} ; PWD=%{DATA:[system][auth][sudo][pwd]} ; USER=%{DATA:[system][auth][sudo][user]} ; COMMAND=%{GREEDYDATA:[system][auth][sudo][command]}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} groupadd(?:\[%{POSINT:[system][auth][pid]}\])?: new group: name=%{DATA:system.auth.groupadd.name}, GID=%{NUMBER:system.auth.groupadd.gid}",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} useradd(?:\[%{POSINT:[system][auth][pid]}\])?: new user: name=%{DATA:[system][auth][user][add][name]}, UID=%{NUMBER:[system][auth][user][add][uid]}, GID=%{NUMBER:[system][auth][user][add][gid]}, home=%{DATA:[system][auth][user][add][home]}, shell=%{DATA:[system][auth][user][add][shell]}$",
                  "%{SYSLOGTIMESTAMP:[system][auth][timestamp]} %{SYSLOGHOST:[system][auth][hostname]} %{DATA:[system][auth][program]}(?:\[%{POSINT:[system][auth][pid]}\])?: %{GREEDYMULTILINE:[system][auth][message]}"] }
        pattern_definitions => {
          "GREEDYMULTILINE"=> "(.|\n)*"
        }
        remove_field => "message"
      }
      date {
        match => [ "[system][auth][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
      geoip {
        source => "[system][auth][ssh][ip]"
        target => "[system][auth][ssh][geoip]"
      }
    }
    else if [fileset][name] == "syslog" {
      grok {
        match => { "message" => ["%{SYSLOGTIMESTAMP:[system][syslog][timestamp]} %{SYSLOGHOST:[system][syslog][hostname]} %{DATA:[system][syslog][program]}(?:\[%{POSINT:[system][syslog][pid]}\])?: %{GREEDYMULTILINE:[system][syslog][message]}"] }
        pattern_definitions => { "GREEDYMULTILINE" => "(.|\n)*" }
        remove_field => "message"
      }
      date {
        match => [ "[system][syslog][timestamp]", "MMM  d HH:mm:ss", "MMM dd HH:mm:ss" ]
      }
    }
  }
}
```


```
sudo vi /etc/logstash/conf.d/30-elasticsearch-output.conf
```
Thêm vào các dòng sau
```
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```
Test file cấu hình logstash
```
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```
Nếu file cấu hình thành công thì khởi động dịch vụ logstash
```
sudo systemctl start logstash
sudo systemctl enable logstash
```
# 4. Cài đặt và cấu hình Filebeat
* Filebeat: collects and ships log files.
* Metricbeat: collects metrics from your systems and services.
* Packetbeat: collects and analyzes network data.
* Winlogbeat: collects Windows event logs.
* Auditbeat: collects Linux audit framework data and monitors file integrity.
* Heartbeat: monitors services for their availability with active probing
## Cài đặt filebeat
```
sudo yum install filebeat
```
Mở file cấu hình filebeat
```
sudo vi /etc/filebeat/filebeat.yml
```
Tìm và Command những dòng
```
...
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["localhost:9200"]
...
```
Và Uncommand những dòng logstash output
```
output.logstash:
  # The Logstash hosts
  hosts: ["localhost:5044"]
 ```

 ```
 sudo filebeat modules enable system
 ```
Xem danh sách các module đã bật hoặc tắt
 ```
 sudo filebeat modules list
 ```
 ### Load template
 ```
 sudo filebeat setup --template -E output.logstash.enabled=false -E 'output.elasticsearch.hosts=["localhost:9200"]'
```

```
sudo filebeat setup -e -E output.logstash.enabled=false -E output.elasticsearch.hosts=['localhost:9200'] -E setup.kibana.host=localhost:5601
```
### Enable filebeat
```
sudo systemctl start filebeat
sudo systemctl enable filebeat
```
```
curl -X GET 'http://localhost:9200/filebeat-*/_search?pretty'
```

# 5. Exploring Kibana Dashboards

Contact me: [Lưu Đình Mác]("fb.com/luudinhmac49")
