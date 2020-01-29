+++
date = "2020-01-29T10:38:34Z"
title = "Setting up prometheus in Docker"
type = "post"
+++

[Prometheus](https://prometheus.io) is a powerful metrics and monitoring system. This is a minimal docker recipe for setting up prometheus locally. Having a dev prometheus server is great because you can test PromQL queries against new metrics.

First let's set up a prometheus configuration file, `prometheus.yaml`.

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'my-test-server'
    static_configs:
      - targets: ['localhost:9090']
```

This file declares all of the scrape targets that prometheus will poll metrics against. The targets are also known as exporters and typically serve an HTTP endpoint under `/metrics` or `/_metrics`. These endpoints respond with metrics data.

Next, start a prometheus container with the configuration file mounted and listening on port :9092

```
$ docker run -d -p 9092:9090 -v $(pwd)/prometheus.yaml:/etc/prometheus/prometheus.yml:ro prom/prometheus
8a73196077d0f3b44655c2a1632a6d6517bbc78ad5156ea2b8c4c14d88059b4a
```

Now you can browse the prometheus frontend and write queries!

```
$ open http://localhost:9092
```

<a href="/prom.png" target="_blank"><img src="/prom.png" style="width: 100%"/></a>
