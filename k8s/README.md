Role Name
=========

# inventory example
```python
[k8s-test]
k8s-zrq01  ansible_ssh_host=10.10.152.39  ansible_ssh_port=60000
k8s-zrq02  ansible_ssh_host=10.10.149.224 ansible_ssh_port=60000
k8s-zrq03  ansible_ssh_host=10.10.39.45   ansible_ssh_port=60000
k8s-zrq04  ansible_ssh_host=10.10.215.69  ansible_ssh_port=60000
k8s-zrq05  ansible_ssh_host=10.10.231.99  ansible_ssh_port=60000
k8s-zrq06  ansible_ssh_host=10.10.214.129 ansible_ssh_port=60000
[etcd]
etcd01  ansible_ssh_host=10.10.152.39  ansible_ssh_port=60000
etcd02  ansible_ssh_host=10.10.149.224 ansible_ssh_port=60000
etcd03  ansible_ssh_host=10.10.39.45   ansible_ssh_port=60000
```
============================================================================
# 环境
* Centos 7 
* Ansible 
* Repository 仓库(k8s安装包和etcd等相关文件, 部署仓库请参考repo docker-compose 部署)
============================================================================
# 部署 仓库
## repo docker 部署 
```python
# docker-compose.yml
[root@10-10-97-127 nginx]# tree
.
├── config
│   └── nginx.conf
└── docker-compose.yaml

1 directory, 2 files
```
```python
# config/nginx.conf
worker_processes  auto;
events {
    worker_connections  65535;
    use epoll;
}

http {
    include       mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /data/nginx/logs/access.log  main;
    open_file_cache max=65535 inactive=20s;
    open_file_cache_min_uses 1;
    open_file_cache_valid 30s;

    sendfile        on;
    #tcp_nopush     on;
    keepalive_timeout  65;

    gzip  on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;

    server {
        listen       80;
        server_name  repo.galaxy.test;
        root /data/repository;
        location / {
            try_files $uri $uri/ =404;
        }

   }
}
```

```python
# docker-compose.yaml
version: '3'
services:
  repository:
    image: nginx:alpine
    restart: always
    ports:
      - "80:80"
    volumes:
      - "./config/nginx.conf:/etc/nginx/nginx.conf:ro"
      - "/data/repository/:/data/repository/"
      - "/data/nginx/logs/:/data/nginx/logs/"

```
mkdir -p /data/repository 
mkdir -p /data/nginx/logs/

## /data/repository 目录结构
```python
# 将相关的安装二进制文件 拷贝到目录.
├── etcd
│   ├── v3.2.18
│   │   ├── etcd
│   │   └── etcdctl
│   └── v3.2.25
│       ├── etcd
│       └── etcdctl
├── flannel
│   └── v0.10.0
│       ├── flanneld
│       └── mk-docker-opts.sh
├── kubernetes
│   └── v1.12.1
│       └── server
│           └── bin
│               ├── apiextensions-apiserver
│               ├── cloud-controller-manager
│               ├── hyperkube
│               ├── kubeadm
│               ├── kube-apiserver
│               ├── kube-controller-manager
│               ├── kubectl
│               ├── kubelet
│               ├── kube-proxy
│               ├── kube-scheduler
│               └── mounter

# 启动:docker-compose up -d
```
测试环境域名 repo.galaxy.test (如果没有部署仓库，可以使用repo.zrq.org.cn, 这个比较慢，带宽有限)

========================================================

# 修改变量 var/main.yaml文件，根据自己的环境做响应调整

# 第一步生成 ssl 证书
```python
ansible-playbook -i k8s.yaml -e 'host=k8s-test' -u user -become --tags 'gen-ssl' -v
```

# 第二步 安装etcd 
```python
ansible-playbook -i k8s.yaml -e 'host=etcd' -u user -become --tags 'etcd' -v
```

# 第三步 安装 k8s-master
```python
ansible-playbook -i k8s.yaml -e 'host=k8s' -u user -become --tags 'k8s-master' -v
```

# 第四步 安装flannel
```python
ansible-playbook -i k8s.yaml -e 'host=k8s' -u user -become --tags 'flannel' -v
```

# 第五步安装 k8s-node
```python
ansible-playbook -i k8s.yaml -e 'host=k8s' -u user -become --tags 'k8s-node' -v
```


## * 注意不支持etcd 3.4及其以上的版本
