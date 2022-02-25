---
title: go prometheus
date: 2022-01-27 17:50:31
tags:
---


官网
https://prometheus.io/docs/prometheus/latest/getting_started/



## Prometheus的特点

-   多维度数据模型。
-   灵活的查询语言。
-   不依赖分布式存储，单个服务器节点是自主的。
-   通过基于HTTP的pull方式采集时序数据。
-   可以通过中间网关进行时序列数据推送。
-   通过服务发现或者静态配置来发现目标服务对象。
-   支持多种多样的图表和界面展示，比如Grafana等。

# 通过docker 部署prometheus

### Volumes & bind-mount

Bind-mount your `prometheus.yml` from the host by running:

```
docker run \
    -p 9090:9090 \
    -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \
    prom/prometheus
```

Or bind-mount the directory containing `prometheus.yml` onto `/etc/prometheus` by running:

```
docker run \
    -p 9090:9090 \
    -v /path/to/config:/etc/prometheus \
    prom/prometheus
```

启动命令：
```
"Path": "/bin/prometheus",
        "Args": [
            "--config.file=/etc/prometheus/prometheus.yml",
            "--storage.tsdb.path=/prometheus",
            "--web.console.libraries=/usr/share/prometheus/console_libraries",
            "--web.console.templates=/usr/share/prometheus/consoles"
        ],
```

设置--storage.tsdb.path 存储路径

```
docker run -itd -p 9090:9090 -v /data/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml  -v /data/prometheus/data:/prometheus prom/prometheus
```

必须以root 用户启动：

```shell
 docker run -itd --user root -p 9090:9090 -v /data/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml -v /data/prometheus/data:/data prom/prometheus  --config.file=/etc/prometheus/prometheus.yml --storage.tsdb.path=/data --web.console.libraries=/usr/share/prometheus/console_libraries --web.console.templates=/usr/share/prometheus/consoles
```

配置：

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["127.0.0.1:9090"]
  - job_name: 'fortress'
    static_configs:
      - targets: ["10.41.145.101:8000"]
        labels:
          instance: fortress
  - job_name: node
    static_configs:
      - targets: ['10.41.145.101:9100']
  - job_name: 'kafka_exporter'
    static_configs:
      - targets: ["10.41.145.101:9308"]
```



计算Counter指标增长率

increase(v range-vector)函数是PromQL中提供的众多内置函数之一。其中参数v是一个区间向量，increase函数获取区间向量中的第一个后最后一个样本并返回其增长量。因此，可以通过以下表达式Counter类型指标的增长率：

increase(node_cpu[2m]) / 120

这里通过node_cpu[2m]获取时间序列最近两分钟的所有样本，increase计算出最近两分钟的增长量，最后除以时间120秒得到node_cpu样本在最近两分钟的平均增长率。并且这个值也近似于主机节点最近两分钟内的平均CPU使用率。

除了使用increase函数以外，PromQL中还直接内置了rate(v range-vector)函数，rate函数可以直接计算区间向量v在时间窗口内平均增长速率。因此，通过以下表达式可以得到与increase函数相同的结果：

1

rate(node_cpu[2m])



Copied!

需要注意的是使用rate或者increase函数去计算样本的平均增长速率，容易陷入“长尾问题”当中，其无法反应在时间窗口内样本数据的突发变化。 例如，对于主机而言在2分钟的时间窗口内，可能在某一个由于访问量或者其它问题导致CPU占用100%的情况，但是通过计算在时间窗口内的平均增长率却无法反应出该问题。

为了解决该问题，PromQL提供了另外一个灵敏度更高的函数irate(v range-vector)。irate同样用于计算区间向量的计算率，但是其反应出的是瞬时增长率。irate函数是通过区间向量中最后两个样本数据来计算区间向量的增长速率。这种方式可以避免在时间窗口范围内的“长尾问题”，并且体现出更好的灵敏度，通过irate函数绘制的图标能够更好的反应样本数据的瞬时变化状态。

1

irate(node_cpu[2m])



Copied!

irate函数相比于rate函数提供了更高的灵敏度，不过当需要分析长期趋势或者在告警规则中，irate的这种灵敏度反而容易造成干扰。因此在长期趋势分析或者告警中更推荐使用rate函数。