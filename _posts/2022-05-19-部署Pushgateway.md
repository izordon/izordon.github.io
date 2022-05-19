---
layout:     post
title:      部署Pushgateway
subtitle:   Pushgateway
date:       2022-05-19
author:     Zordon Yang
header-img: img/post-bg-YesOrNo.jpg
catalog: false
tags:
    - 部署文档
    - Prometheus

- [Pushgateway](#pushgateway)
  - [部署](#部署)
    - [1、下载Pushgateway](#1下载pushgateway)
    - [2、以服务方式启动](#2以服务方式启动)
    - [3、启动服务，设置开机自启](#3启动服务设置开机自启)
    - [4、检查9091端口](#4检查9091端口)
    - [5、关于Pushgateway的使用](#5关于pushgateway的使用)
    - [6、可通过python library，使用push 方式把数据推送到pushgateway，例如：](#6可通过python-library使用push-方式把数据推送到pushgateway例如)
  - [Pushgateway使用相关](#pushgateway使用相关)
  - [附件📎](#附件)

# Pushgateway

## 部署

### 1、下载Pushgateway

```bash
wget https://github.com/prometheus/pushgateway/releases/download/v1.4.0/pushgateway-1.4.0.linux-amd64.tar.gz 
tar -zxvf pushgateway-1.4.0.linux-amd64.tar.gz 
mv pushgateway-1.4.0.linux-amd64 /usr/local/pushgateway-1.4.0 
cp -r /usr/local/pushgateway-1.4.0/pushgateway /usr/local/bin/ 
```

### 2、以服务方式启动 

```bash
vi /usr/lib/systemd/system/pushgateway.service 
```

```shell
[Unit] 
Description=pushgateway 
Documentation=https://github.com/prometheus/pushgateway 
After=network.target 


[Service] 
Type=simple 
ExecStart=/usr/local/bin/pushgateway \ 
          #设置持久化文件，防止pushgateway挂掉，数据丢失 
          --persistence.file=/usr/local/pushgateway-1.4.0/pushgateway.data \ 
          #每5m上传一次数据 
          --persistence.interval=5m \ 
          > /dev/null 2>&1 
Restart=on-failure 


[Install] 
WantedBy=multi-user.target 
```

### 3、启动服务，设置开机自启 

```bash
systemctl daemon-reload 
systemctl enable pushgateway 
systemctl start pushgateway 
```

### 4、检查9091端口 

```bash
netstat -anlptu|grep 9091 
```

```output
tcp6       0      0 :::9091              :::*               LISTEN      5216/pushgetway 
```

### 5、关于Pushgateway的使用

仅能传递数值到Pushgateway，例如："timeout 15”；像"error 连接失败"和"error timeout"都无法上传。 

### 6、可通过python library，使用push 方式把数据推送到pushgateway，例如： 

```python
coding=utf-8 

import sys 
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway 
#设置utf-8，否则会编码报错 
if sys.getdefaultencoding() != 'utf-8': 
    reload(sys) 
    sys.setdefaultencoding('utf-8') 

registry = CollectorRegistry() 
http_status_code = '200' 
api_status_code = '0' 
err_msg = '成功' 

#若不填写instance，Pushgateway会显示为instance="" 
g = Gauge('api_status_code', '检测API接口返回值',['err_msg','instance'], registry=registry) 
g.labels(err_msg,'问政平台工单查询接口').set(api_status_code) 

g = Gauge('http_status_code', '检测API接口http返回值',['instance'], registry=registry) 
g.labels('问政平台工单查询接口').set(http_status_code) 
push_to_gateway('172.19.1.244:9091', job='check_http_status', registry=registry) 
```

## Pushgateway使用相关

部署完成Pushgateway后，配置好prometheus配置文件中自定义监控项，grafana导入json模版即可（模版可参照Grafana.md中的附件json）。

## 附件📎

**检查api接口连通性的shell脚本：check_api.sh**

```shell
#!/bin/bash

#form表单提交
form_com="curl -s -H 'Content-Type':'application/x-www-form-urlencoded' -XPOST -d"

#return http code
curl="curl -Is -XPOST -w %{http_code} -o /dev/null"

#upload data
run="python /home/shell/api_monitor/upload.py"

#问政平台工单查询接口
case_1()
{
url="http://19.53.17.18:8089/xzwz/interface/szcg/form/query.do"
http_code=`${curl} ${url}`
output=`${form_com} 'jsonParam=1fc5d514d79d58a9f7e16902245df288f25cb2d2bb3a4005b7626fd727c29d80' ${url} | sed 's/^.//g' | sed 's/.$//g'`
info="\"job\":\"问政平台工单查询接口-原始接口\",\"instance\":\"${url}\",\"http_status_code\":\"${http_code}\","
echo "{${info}${output}}" | ${run}
}

#市数管核查结果拉取接口
case_2()
{
url="http://19.48.25.47:8080/KDUMExchangeServer/xzq/findEventCheckPage"
http_code=`${curl} ${url}`
output=`curl -s -H 'Content-Type':'application/json' -XPOST -d "{\"va_status\":0,\"page\": 0,\"register_time_end\": \"2020-10-20 18:44:52\",\"register_time_start\": \"2020-10-20 18:38:52\",\"rows\": 1000,\"token\": \"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2MDcxMzkxNzg3NzUsInBheWxvYWQiOiJ7XCJsYXN0VGltZVwiOlwiMjAyMC0xMi0wNCAxMTozMjo1OFwiLFwiZGlzdHJpY3RJZFwiOlwiMDc0ODg2MjgtNjZGNy0xMTQ5LUJDMDctQzM0MkRDNDcwQjg3XCIsXCJsb2dpblB3ZFwiOlwia2luZ2RvbVwiLFwiYWRkaXRpb25hbFJvbGVJZHNcIjpcIkU2NUYxQzA3LUIwQjYtREQxMS04MDA5LTAwMUU5MEEyMEEwQ1wiLFwibG9naW5OYW1lXCI6XCJraW5nZG9tXCIsXCJkZXB0SWRcIjpcIkI0NUMxQzA3LUIwQjYtREQxMS04MDA5LTAwMUU5MEEyMEEwQ1wifSJ9.ma5kzuZrwecANGLC_r6zB0mGlTA4EeoU9IFIYsezscI\",\"user_id\": \"0000000073ecbe76017434826e1810aa\"}" ${url} | sed 's/^.//g' | sed 's/.$//g'`
info="\"job\":\"市数管核查结果拉取接口-原始接口\",\"instance\":\"${url}\",\"http_status_code\":\"${http_code}\","
echo "{${info}${output}}" | ${run}
}

case_1
case_2
```

**上传检查结果的python脚本：upload.py**

```python
# coding=utf-8
import sys,json
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

reload(sys)
sys.setdefaultencoding('utf8')
registry = CollectorRegistry()

def upload_data():
    
    g = Gauge('api_status_code', '检测API接口返回值',['err_msg','instance'], registry=registry)
    g.labels(msg,ins).set(api_code)

    g = Gauge('http_status_code', '检测API接口http返回值',['instance'], registry=registry)
    g.labels(ins).set(http_code)

    push_to_gateway('pushgateway_url:9091',job=str(name) , registry=registry)


if __name__ == '__main__':
    dict_str = json.loads(sys.stdin.read())

    if 'errcode' in dict_str :
        api_code = dict_str["errcode"]
    else:
        api_code = dict_str["code"]

    if 'errmsg' in dict_str :
        msg = dict_str["errmsg"]
    else:
        msg = dict_str["msg"]

    name = dict_str["job"]
    ins = dict_str["instance"]
    http_code = dict_str["http_status_code"]
    upload_data()
```