# grpc



## 1. prometheus 监控直方图

[prometheus文档](https://prometheus.io/docs/prometheus/latest/getting_started/)
[grpc_prometheus学习文档](https://hant.helplib.com/GitHub/article_139676)

可以使用 `grpc_prometheus.EnableHandlingTimeHistogram` 开启直方图统计, 度量rpc延遲分佈的一種很好的方法

所有**伺服器端**指標以 `grpc_server` 作為Prometheus子系統 NAME。 所有**客戶端**度量都以 `grpc_client開始`

可以使用 `prometheus+grafana` 进行展示

`prometheus`自身擔任**database**的角色，搭配`grafana`我們可以可視化的監控各種時間序列數據

### 1.1 自定义参数

``` text
Counter: 重點方法inc ，一個累計型的metric
CounterVec: Counter支援Label
Gauge: 重點方法set，自己設定各種value 最常用
GaugeVec: Gauge支援Label
Histogram: 重點方法Observe，集計型的metric
HistogramVec: Histogram支援Label
Summary: 重點方法Observe，集計型的metric
SummaryVec: Summary支援Label
```

有Label的話在使用grafana的時候可以設定一些條件來過濾