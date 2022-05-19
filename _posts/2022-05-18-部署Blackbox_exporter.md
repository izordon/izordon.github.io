---
layout:     post
title:      部署Blackbox_exporter
subtitle:   Blackbox_exporter
date:       2022-05-18
author:     Zordon Yang
header-img: img/post-bg-YesOrNo.jpg
catalog: false
tags:
    - 部署文档
    - Prometheus
---
- [Blackbox_exporter](#blackbox_exporter)
  - [部署](#部署)
    - [1、下载Blackbox_exporter](#1下载blackbox_exporter)
    - [2、配置以服务方式启动](#2配置以服务方式启动)
    - [4、设置开机自启&启动服务](#4设置开机自启启动服务)
    - [5、测试访问](#5测试访问)
    - [6、Blackbox_exporter配置文件](#6blackbox_exporter配置文件)
    - [7、Prometheus配置](#7prometheus配置)
  - [Blackbox_exporter使用相关](#blackbox_exporter使用相关)

# Blackbox_exporter

## 部署

### 1、下载Blackbox_exporter

```
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.18.0/blackbox_exporter-0.18.0.linux-amd64.tar.gz
tar xf blackbox_exporter-0.18.0.linux-amd64.tar.gz -C /usr/local/
mv /usr/local/blackbox_exporter-0.18.0.linux-amd64 /usr/local/blackbox_exporter
cp -r /usr/local/blackbox_exporter/blackbox_exporter /usr/local/bin/
cd /usr/local/blackbox_exporter
```

### 2、配置以服务方式启动 

添加到系统服务，方便于管理 

```bash
vi /etc/systemd/system/blackbox_exporter.service 
```

```sh
[Unit] 
Description=blackbox_exporter
Documentation=https://github.com/prometheus/blackbox_exporter
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter \
          --config.file=/usr/local/blackbox_exporter/blackbox.yml

Restart=on-failure

[Install] 
WantedBy=multi-user.target 
```

### 4、设置开机自启&启动服务

```bash
systemctl daemon-reload 
systemctl enable blackbox_exporter 
systemctl start blackbox_exporter 
```

### 5、测试访问

```url
http://IP:9115
```

### 6、Blackbox_exporter配置文件

```yaml
modules:
  http_2xx:
    prober: http
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      method: GET
      preferred_ip_protocol: "ip4"
      ip_protocol_fallback: false
      no_follow_redirects: false
      fail_if_ssl: false
      fail_if_not_ssl: false
      tls_config:
        insecure_skip_verify: true
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
```

### 7、Prometheus配置

```
参照Prometheus.md中web配置相关字段
```

## Blackbox_exporter使用相关

部署完成Blackbox_exporter后，配置好prometheus配置文件中web监控项，grafana导入json模版即可（模版可参照Grafana.md中的附件json）。

