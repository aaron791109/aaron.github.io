---
layout: post
title:  "How to Monitor Elasticsearch with prometheus operator"
date:   2022-10-23
categories: ''
---
因工作需求之前已經安裝了`prometheus operator`
與原先的 `Prometheus` 差異大概就是配置變成要透過 CRD 去傳遞即可

# Install Elasticsearch Plugins
我們選用 vvanholl 維護的專案
選擇與你的 ES 集群相對應版本的exporter
在你的 `Deployments` 或 `StatefulSets` 添加一個安裝插件使用的 `initContainer`
`記得需要掛載Volume`
```YAML
initContainers:
       - name: install-plugins
         command:
         - sh
         - -c
         - |
           bin/elasticsearch-plugin install -b https://github.com/vvanholl/elasticsearch-prometheus-exporter/releases/download/${VERSION}/prometheus-exporter-${VERSION}.zip
          volumeMounts:
            - name: plugins
              mountPath: /usr/share/elasticsearch/plugins
      containers:
        - name: elasticsearch
          ...
          volumeMounts:
            - name: plugins
              mountPath: /usr/share/elasticsearch/plugins

```
安裝完成後，如有安裝 `Kibana` 至 devtools 查看或直接使用`curl`
```bash
GET /_prometheus/metrics
curl ${IP}:${PORT}/_prometheus/metrics
```
# Setting Prometheus-operator ServiceMonitor
需要額外從 `Secret` 拿到帳號密碼
帳號密碼需要使用 base64 加密過才能填入
範例
```bash
echo -n "username" | base64
```
```YAML
apiVersion: v1
kind: Secret
metadata:
  name: es-creds
  namespace: monitoring
data:
  username: "base64 加密過後的字串"
  password: "base64 加密過後的字串"
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: elasticsearch-metrics
  namespace: monitoring

spec:
  selector:
    matchLabels:
      app: elasticsearch-master  # 你的service該有的label
  endpoints:
    - interval: 60s   # 輪詢時間
      path: "/_prometheus/metrics"
      port: http      # es提供服務的port name
      scheme: https   # 這邊我有開 xpack 所以需要使用https
      tlsConfig:      # 因為xpack 開的setting
        insecureSkipVerify: true
      basicAuth:
        password:
          name: es-creds # 指的是去Secret的es-creds
          key: password  # es-creds裡面對應的key
        username:
          name: es-creds
          key: username
  namespaceSelector:  #es 在哪個namespace
    matchNames:
    - logging
```
# Grafana Dashboard
[here on grafana.com](https://grafana.com/grafana/dashboards/266-elasticsearch/)


# 最小權限原則
在 `Elasticsearch` 上創建角色並附加到用戶上
賦予的權限為
```bash
Cluster privileges: monitor ,monitor_snapshot

Index privileges: indices = * , privileges = monitor
```
# Done
這樣我們就完成了使用 `ServiceMonitor` 來監控 `Elasticsearch` 的數據變化
