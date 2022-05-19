---
layout:     post
title:      部署Node_exporter
subtitle:   Node_exporter
date:       2022-05-18
author:     Zordon Yang
header-img: img/post-bg-YesOrNo.jpg
catalog: false
tags:
    - 部署文档
    - Prometheus
---
- [node_exporter](#node_exporter)
  - [部署](#部署)
    - [1、下载node_eporter](#1下载node_eporter)
    - [2、以服务方式启动](#2以服务方式启动)
    - [3、启动服务，设置开机自启](#3启动服务设置开机自启)
    - [4、检查9100端口](#4检查9100端口)
  - [node_exporter使用相关](#node_exporter使用相关)

# node_exporter

## 部署

### 1、下载node_eporter 

```bash
wget wget https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz 
tar -xf node_exporter-0.18.1.linux-amd64.tar.gz 
mv node_exporter-0.18.1 /usr/local/node_exporter-0.18.1 
cp /usr/loca/node_exporter-0.18.1/node_exporter /usr/local/bin/ 
```

### 2、以服务方式启动 

```bash
vim /usr/lib/systemd/system/node_exporter.service 
```

```shell
[Unit] 
Description=node_exporter 
Documentation=https://github.com/prometheus/node_exporter 
After=network.target 

[Service] 
Type=simple 
ExecStart=/usr/local/bin/node_exporter
Restart=on-failure 

[Install] 
WantedBy=multi-user.target 
```

### 3、启动服务，设置开机自启 

```bash
systemctl daemon-reload 
systemctl enable node_exporter 
systemctl start node_exporter 
```

### 4、检查9100端口 

```bash
netstat -anlptu|grep 9100 
```

```output
tcp6       0      0 :::9100            :::*           LISTEN      5216/node_exporter 
```

## node_exporter使用相关

部署完成node_exporter后，配置好prometheus配置文件中服务器节点监控项，grafana导入json模版即可（模版可参照Grafana.md中的附件json）。