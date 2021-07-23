# Logstash

![basic logstash pipeline](..\img\basic_logstash_pipeline.png)



## Input

### Elastic Beat框架直连

开启5044监听，让filebeat等beat框架连接进来并且output到es

```json
input {
  beats {
    port => 5044
  }
}

output {
  elasticsearch {
    hosts => "localhost:9200"
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}" 
    document_type => "%{[@metadata][type]}" 
  }
}
```

### 其他input插件

如kafka、redis等

## Filter

### grok

格式化日志

```
55.3.244.1 GET /index.html 15824 0.043
```

当input是直接的log时

```json
input {
      file {
        path => "/var/log/http.log"
      }
    }
    filter {
      grok {
        match => { "message" => "%{IP:client} %{WORD:method} %{URIPATHPARAM:request} %{NUMBER:bytes} %{NUMBER:duration}" }
      }
    }
```

在 grok 过滤器之后，会有一些额外的字段

- `client: 55.3.244.1`

- `method: GET`
- `request: /index.html`
- `bytes: 15824`
- `duration: 0.043`

### mutate

对事件字段执行一般转换。您可以重命名、删除、替换和修改事件中的字段。 

### drop

完全丢弃一个事件，例如调试事件。 

## Output

### elasticsearch

将事件数据发送到 Elasticsearch

### file

将事件数据写入磁盘上的文件
