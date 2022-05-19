---
layout:     post
title:      部署Alertmanager
subtitle:   Alertmanager
date:       2022-05-19
author:     Zordon Yang
header-img: img/post-bg-YesOrNo.jpg
catalog: false
tags:
    - 部署文档
    - Prometheus
---
- [Alertmanager](#alertmanager)
  - [部署](#部署)
    - [1、下载alertmanager](#1下载alertmanager)
    - [2、以服务方式启动](#2以服务方式启动)
    - [3、启动服务，设置开机自启](#3启动服务设置开机自启)
    - [4、配置文件参考(webhook)](#4配置文件参考webhook)

# Alertmanager

## 部署

### 1、下载alertmanager 

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.19.0/alertmanager-0.19.0.linux-amd64.tar.gz 
tar zxvf alertmanager-0.19.0.linux-amd64.tar.gz 
mv alertmanager-0.19.0.linux-amd64 /usr/local/alertmanager-0.19.0 
cp -r /usr/local/alertmanager-0.19.0/alertmanager /usr/local/bin/ 
```

### 2、以服务方式启动 

```bash
vi /usr/lib/systemd/system/alertmanager.service 
```

```shell
[Unit]
Description=alertmanager
Documentation=https://github.com/prometheus/alertmanager
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/alertmanager --config.file=/usr/local/alertmanager-0.19.0/alertmanager.yml
Restart=on-failure
[Install]
WantedBy=multi-user.target
```

### 3、启动服务，设置开机自启 

```bash
systemctl daemon-reload && systemctl start alertmanager && systemctl enable alertmanager 
```

### 4、配置文件参考(webhook) 

```bash
cat alertmanager.yml 
```

```yaml
global:
  resolve_timeout: 5m #在报警恢复的时候不是立马发送消息的，在接下来的这个时间内，如果没有此报警信息触发，才发送报警恢复消息
  smtp_from: 'yangzudong@fastersoft.com.cn' #发件邮箱地址
  smtp_smarthost: 'smtp.exmail.qq.com:465' #腾讯企业邮箱smtp服务器
  smtp_auth_username: 'yangzudong@fastersoft.com.cn' #邮箱地址
  smtp_auth_password: '邮箱授权码'
  smtp_require_tls: false #关闭tls认证
  smtp_hello: 'fastersoft.com.cn'

route:
  group_by: ['alertname'] #路由分组标签
  group_wait: 10s #当传入的警报创建了一组新的警报时，请至少等待多少秒发送初始通知
  group_interval: 10s  #发送第一个通知时，请等待多少分钟发送一批已开始为该组触发的新警报
  repeat_interval: 1h #如果警报已成功发送，请等待多少小时以重新发送警报
  receiver: 'wechat_webhook' #此处表示使用wechat_webhook路由

receivers:
- name: 'wechat_webhook'
  webhook_configs: #webhook地址
  - url : 'webhook_url'
    send_resolved: true #是否发送恢复消息

- name: 'email' #邮件告警配置
  email_configs:
  - to: 'yangzudong@fastersoft.com.cn'
    send_resolved: true
    html: '{{ template "email.to.html" . }}'
    headers: { Subject: " {{ .CommonLabels.instance }} {{ .CommonAnnotations.summary }}" }

inhibit_rules:
  - source_match:
      severity: 'critical' #告警级别
    target_match:
      severity: 'warning' #告警级别
    equal: ['alertname', 'dev', 'instance']
```

