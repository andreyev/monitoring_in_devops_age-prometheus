---
[comment]: to build just run `docker run --rm -v $PWD:/home/marp/app/ -e MARP_USER="$(id -u):$(id -g)" -e LANG=$LANG marpteam/marp-cli *.md`
theme: default
_class: lead
paginate: true
backgroundColor: #ADD8E6

---

![bg left:40% 80%](https://upload.wikimedia.org/wikipedia/commons/3/38/Prometheus_software_logo.svg)

# **Monitoramento na era DevOps**

## Uma introdução ao Prometheus

(Repositório do hands-on em https://github.com/andreyev/prometheus_hands-on)
![](https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=https://github.com/andreyev%2Fprometheus_hands-on)

---

# _~~whoami~~_
# ~~história~~

_Can we skip to the good part?_

---

# Observabilidade

- logs
- rastreabilidade
- métricas

---

# Observabilidade (analogia)

- **logs**: variação no pedal do acelerador, pista em declive, troca de marcha
- **rastreabilidade**: variação no consumo de combustível, consumo por quilômetro
- **métricas**: leitura do velocimetro

---

# Observabilidade

- ~~logs~~
- ~~rastreabilidade~~
- **métricas**

---

# Métricas e amostras

```
up{instance="demo.do.prometheus.io:9090", job="prometheus"} 1
```

---

# Métricas e amostras

nome da métrica|up
:--------------|:---------------------
**labels**     |**{ instance="demo.do.prometheus.io:9090", job="prometheus" }**
**valor**      |**1**

---

# Filosofia Unix

> _“This is the Unix philosophy: write programs that do one thing and do it well. Write programs to work together. Write programs that handle text streams, because that is a universal interface.”_ —Douglas McIlroy.

---

# Filosofia Unix

```
$ ps -ef  | grep xyz
```

---

# Filosofia Unix
## Arquitetura básica

![](https://prometheus.io/assets/tutorial/architecture.png)

---

# Filosofia Unix

- alertmanager
- pushgateway
- exporters (blackbox exporter, node-exporter etc)
- thanos
- grafana?!
- curl
---

# Filosofia Unix
## Arquitetura completa

![](https://blog.palark.com/wp-content/uploads/2023/10/prometheus-architecture-1024x521.png)

---

# Pilares
## PromQL

> _"One ~~ring~~query to rule them all"_

---

# Pilares
## PromQL

> _"One ~~ring~~query to rule them all"_

- queries
- API
- grafana
- alert rules expr

---

# Pilares
## PromQL

```
node_filesystem_device_error == 1
```

---

# Pilares
## PromQL

Notação matemática convencional

```
((node_filesystem_avail_bytes * 100) / node_filesystem_size_bytes < 10 and
ON (instance, device, mountpoint) predict_linear(node_filesystem_avail_bytes{fstype!~"tmpfs"}[1h], 24 * 3600) < 0 and
ON (instance, device, mountpoint) node_filesystem_readonly == 0) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
```

---

# Pilares
## PromQL: regex :warning:

RE2 vs PCRE

---

# Pilares
## PromQL
### API

```
$ curl -s 'http://localhost:9090/api/v1/query?query=up' | jq '.data.result[] | {instance: .metric.instance, status: .value[1]}'
{
  "instance": "node-exporter:9100",
  "status": "1"
}
{
  "instance": "https://www.google.com",
  "status": "1"
}
{
  "instance": "demo:8001",
  "status": "1"
}
{
  "instance": "prometheus:9090",
  "status": "1"
}
{
  "instance": "pushgateway:9091",
  "status": "1"
}
{
  "instance": "alertmanager:9093",
  "status": "1"
}
```
---

# Pilares
## PromQL

Bibliotecas e exemplos: <https://samber.github.io/awesome-prometheus-alerts>

---

# Pilares
## Fontes de dados

---

# Pilares
## Fontes de dados

- Exporters
- Targets
- Jobs
- Clientes e bibliotecas

---

# Pilares
## Fontes de dados

- **Exporters**: apps que colhem e servem dados de outros apps. Mysql-exporter, por exemplo.
- **Targets**: alvo (geralmente URL) de uma fonte de dados (aka uma app instrumentada ou um exporter).
- **Jobs**: Agrupadores de tipos de targets pelas suas características semelhantes (_path_, porta, tipo de app etc)
- **Clientes e bibliotecas**: _next_


---

# Pilares
## Exporters

- Node exporter: info(RAM, disco CPU etc) do computador em que ele está sendo executado
- Blackbox exporter: HTTP, HTTPS, DNS, TCP, ICMP etc.
- Cadvisor: "node exporter" para containeres.
- Pushgateway: cache para métricas assíncronas, curtas ou temporárias, como cronjobs, scripts ou pipelines de CI/CD.
- mysql, postgres, elastic search, RDS, cloudwatch, JIRA, kafka, RabbitMQ, Varnish etc ( ~200 "oficiais/suportados" em https://prometheus.io/docs/instrumenting/exporters/)

---

# Pilares
## Clientes e bibliotecas - Métricas

```
$ curl -s http://localhost:8001/metrics | head
# HELP python_gc_objects_collected_total Objects collected during gc
# TYPE python_gc_objects_collected_total counter
python_gc_objects_collected_total{generation="0"} 249.0
python_gc_objects_collected_total{generation="1"} 12.0
python_gc_objects_collected_total{generation="2"} 0.0
# HELP python_gc_objects_uncollectable_total Uncollectable objects found during GC
# TYPE python_gc_objects_uncollectable_total counter
python_gc_objects_uncollectable_total{generation="0"} 0.0
python_gc_objects_uncollectable_total{generation="1"} 0.0
python_gc_objects_uncollectable_total{generation="2"} 0.0
```
---

# Pilares
## Clientes e bibliotecas - Instrumentação

```
from flask import Flask, jsonify, request
from prometheus_client import make_wsgi_app, Counter, Histogram
from werkzeug.middleware.dispatcher import DispatcherMiddleware
import time
app = Flask(__name__)
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {
    '/metrics': make_wsgi_app()
})
REQUEST_LATENCY = Histogram(
    'app_request_latency_seconds',
    'Application Request Latency',
    ['method', 'endpoint']
)
@app.route('/')
def hello():
    start_time = time.time()
response = jsonify(message='Hello, world!')
    REQUEST_LATENCY.labels('GET', '/').observe(time.time() - start_time)
    return response
if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
---

# Pilares
## SD (_Service Discovery_)

- Arquivo
- Consul
- Nuvem (EC2 etc)
- Estático
- etc

---

# Pilares
## TSDB (_Time Series Data Base_)

---

# Pilares
## TSDB (_Time Series Data Base_)

![](https://training.promlabs.com/static/prometheus-data-model-series-graph-379763ef781612bc7dddf098e32b0525.svg)

---

# Alertmanager (_sneak peek_): integração
![](https://miro.medium.com/v2/resize:fit:640/format:webp/0*vUKS3LWql6z1hKzZ.png)

---

# Alertmanager (_sneak peek_): rotas
![](https://groups.google.com/group/prometheus-developers/attach/2ffe0a4a74643/alertmanager-alertrouting.png?part=0.1&view=1)

---

# Obrigado!!!!!1
Repositório desta apresentação https://github.com/andreyev/monitoring_in_devops_age-prometheus
![](https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=https://github.com/andreyev%2Fmonitoring_in_devops_age-prometheus)
