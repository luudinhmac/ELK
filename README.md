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
