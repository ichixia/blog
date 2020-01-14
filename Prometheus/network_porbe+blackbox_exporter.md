#### 1. Blackbox Exporter  
Blackbox Exporter是Prometheus社区提供的官方黑盒监控解决方案，其允许用户通过：HTTP、HTTPS、DNS、TCP以及ICMP的方式对网络进行探测。用户可以直接使用go get命令获取Blackbox Exporter源码并生成本地可执行文件：  
```
go get prometheus/blackbox_exporter
```
也可以在下面的网址，选择合适自己环境的产品包进行下载：  
```
https://github.com/prometheus/blackbox_exporter/releases
```
我们选择 [blackbox_exporter-0.12.0.linux-amd64.tar.gz](https://github.com/prometheus/blackbox_exporter/releases/download/v0.12.0/blackbox_exporter-0.12.0.linux-amd64.tar.gz) 这个版本进行下载，下载解压后的文件夹是这样的内容：  
```
$ ll

drwxrwxr-x 2 admin admin     4096 Feb 27  2018 ./
drwxr-xr-x 3 root  root      4096 Dec  9 19:21 ../
-rwxr-xr-x 1 admin admin 16074005 Feb 27  2018 blackbox_exporter*
-rw-rw-r-- 1 admin admin      639 Feb 27  2018 blackbox.yml
-rw-rw-r-- 1 admin admin    11357 Feb 27  2018 LICENSE
-rw-rw-r-- 1 admin admin       94 Feb 27  2018 NOTICE
```

主要的就是 **blackbox_exporter**启动程序 和 配置文件 **blackbox.yml**
我们通过下面这个简单的命令就能启动一个Blackbox Exporter：  
```
$ ./blackbox_exporter
```
![image](https://note.youdao.com/yws/public/resource/3a3a66b75507de60516493c2671f64cb/xmlnote/FD04AC6EE42D4C6BB3DC9FFC6A8EA941/6372)  

启动之后我们可以通过访问http://127.0.0.1:9115查看web端。    

![](https://note.youdao.com/yws/public/resource/3a3a66b75507de60516493c2671f64cb/xmlnote/6E5FA778B98D4A0CAD1845CDC57B8117/6374)  

这样就启动了一个Blackbox Exporter  

#### 2. Blackbox Exporter配置信息  

运行Blackbox Exporter时，需要用户提供探针的配置信息，这些配置信息可能是一些自定义的HTTP头信息，也可能是探测时需要的一些TSL配置，也可能是探针本身的验证行为。   

在Blackbox Exporter每一个探针配置称为一个module，并且以YAML配置文件的形式提供给Blackbox Exporter。 每一个module主要包含以下配置内容，包括探针类型（prober）、验证访问超时时间（timeout）、以及当前探针的具体配置项：  
```
  # 探针类型：http、 tcp、 dns、 icmp.
  prober: <prober_string>

  # 超时时间
  [ timeout: <duration> ]

  # 探针的详细配置，最多只能配置其中的一个
  [ http: <http_probe> ]
  [ tcp: <tcp_probe> ]
  [ dns: <dns_probe> ]
  [ icmp: <icmp_probe> ]
```

下面是一个简化的探针配置文件blockbox.yml，包含两个HTTP探针配置项：  
```
modules:
  http_2xx:
    prober: http
    http:
      method: GET
  http_post_2xx:
    prober: http
    http:
      method: POST
```  

通过运行以下命令，并指定使用的探针配置文件启动Blockbox Exporter实例： 
```
blackbox_exporter --config.file=/etc/prometheus/blackbox.yml
```

启动成功后，就可以通过访问http://127.0.0.1:9115/probe?module=http_2xx&target=baidu.com对baidu.com进行探测。这里通过在URL中提供module参数指定了当前使用的探针，target参数指定探测目标，探针的探测结果通过Metrics的形式返回：  
```
# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.011633673
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 0.117332275
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
# HELP probe_http_content_length Length of http content response
# TYPE probe_http_content_length gauge
probe_http_content_length 81
# HELP probe_http_duration_seconds Duration of http request by phase, summed over all redirects
# TYPE probe_http_duration_seconds gauge
probe_http_duration_seconds{phase="connect"} 0.055551141
probe_http_duration_seconds{phase="processing"} 0.049736019
probe_http_duration_seconds{phase="resolve"} 0.011633673
probe_http_duration_seconds{phase="tls"} 0
probe_http_duration_seconds{phase="transfer"} 3.8919e-05
# HELP probe_http_redirects The number of redirects
# TYPE probe_http_redirects gauge
probe_http_redirects 0
# HELP probe_http_ssl Indicates if SSL was used for the final redirect
# TYPE probe_http_ssl gauge
probe_http_ssl 0
# HELP probe_http_status_code Response HTTP status code
# TYPE probe_http_status_code gauge
probe_http_status_code 200
# HELP probe_http_version Returns the version of HTTP of the probe response
# TYPE probe_http_version gauge
probe_http_version 1.1
# HELP probe_ip_protocol Specifies whether probe ip protocol is IP4 or IP6
# TYPE probe_ip_protocol gauge
probe_ip_protocol 4
# HELP probe_success Displays whether or not the probe was a success
# TYPE probe_success gauge
probe_success 1
```

从返回的样本中，用户可以获取站点的DNS解析耗时、站点响应时间、HTTP响应状态码等等和站点访问质量相关的监控指标，从而帮助管理员主动的发现故障和问题。  

#### 与Prometheus集成  
如果要持续地对网络进行探测，仅仅是启动Blackbox Exporter是不够的，要么自己写一个脚本定时地用请求通过Blackbox Exporter对目标进行探测，得到响应结果后再根据结果执行自己不同的后续操作，但这似乎有点麻烦。这个时候就需要集成Prometheus，让它来为我们做这些事情。  

在Prometheus下配置对Blockbox Exporter实例的采集任务即可。最直观的配置方式：  
```
- job_name: baidu_http2xx_probe
  params:
    module:
    - http_2xx
    target:
    - baidu.com
  metrics_path: /probe
  static_configs:
  - targets:
    - 127.0.0.1:9115
- job_name: prometheus_http2xx_probe
  params:
    module:
    - http_2xx
    target:
    - prometheus.io
  metrics_path: /probe
  static_configs:
  - targets:
    - 127.0.0.1:9115
```

这里分别配置了名为baidu_http2x_probe和prometheus_http2xx_probe的采集任务，并且通过params指定使用的探针（module）以及探测目标（target）。  

简而言之就是分别给Blackbox Exporter发送下面的请求去探测target。  
```
# job : baidu_http2xx_probe
http://127.0.0.1:9115/probe?module=http_2xx&target=baidu.com

# job : prometheus_http2xx_probe  
http://127.0.0.1:9115/probe?module=http_2xx&target=prometheus.io
```


那问题就来了，假如我们有N个目标站点且都需要M种探测方式，那么Prometheus中将包含N * M个采集任务，从配置管理的角度来说显然是不可接受的。我们可以用Prometheus的Relabeling能力对这些配置进行简化，配置方式如下： 
```
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://prometheus.io    # Target to probe with http.
        - https://prometheus.io   # Target to probe with https.
        - http://example.com:8080 # Target to probe with http on port 8080.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115
```

这里针对每一个探针服务（如http_2xx）定义一个采集任务，并且直接将任务的采集目标定义为我们需要探测的站点。在采集样本数据之前通过relabel_configs对采集任务进行动态设置。  

* 第1步，根据Target实例的地址，写入__param_target标签中。__param_<name>形式的标签表示，在采集任务时会在请求目标地址中添加<name>参数，等同于params的设置；
* 第2步，获取__param_target的值，并覆写到instance标签中；
* 第3步，覆写Target实例的__address__标签值为BlockBox Exporter实例的访问地址。

通过以上3个relabel步骤，即可大大简化Prometheus任务配置的复杂度:  
![](https://note.youdao.com/yws/public/resource/3a3a66b75507de60516493c2671f64cb/xmlnote/8AB190AAE69C4ED3BAF0A57223E6ED21/6459)  

#### 参考  
[网络探测：Blackbox Exporter](https://yunlzheng.gitbook.io/prometheus-book/part-ii-prometheus-jin-jie/exporter/commonly-eporter-usage/install_blackbox_exporter)  
[blackbox_exporter使用](https://www.li-rui.top/2018/11/23/monitor/blackbox_exporter%E4%BD%BF%E7%94%A8/)

