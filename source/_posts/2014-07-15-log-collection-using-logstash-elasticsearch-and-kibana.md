title: '使用logstash, elasticsearch, kibana做日志分析系统'
date: 2014-07-15 16:09:00
tags: [logstash, elasticsearch, kibina]
---

一般情况下，一个站点会由多台server构成, 每台server每天会产生多种不同的日志，为了方面分析线上日志，常用做法是把线上日志集中存储到一个地方，然后按照日期，日志类型，日志内关键字等条件查询。而logstash, elasticsearch, 和kibana就是用来做这件事的开源解决方案。

[Logstash](http://logstash.net/) 运行于Jruby上的日志收集和分析系统，支持多种后端存储，包括elasticsearch, s3, riak, mongodb, redis等
[Elasticsearch](http://www.elasticsearch.org/) Java写的搜索引擎，简单好用，很适合日志的存储和查询, 简称ES
[Kibana](http://www.elasticsearch.org/overview/kibana/) Elasticsearch的查询前端，最早用PHP写的，后来用Ruby重新实现了一遍，最新版本是纯Javascript的，直接在浏览器端运行。

架构上，在服务器不多的情况下，可以使用Logstash分析日志后，直接存储到ES里面，然后调用Kibana做日志查询。

如果服务器很多，或者是需要跨网络，那么可以使用前向代理，做成树状结构，树叶是Logstash，分析完成后传给树枝[logstash-forwarder](https://github.com/elasticsearch/logstash-forwarder), 然后传给树干Logstash, 最终输出汇总给ES

Logstash, elasticsearch, kibana的安装都很简单，这里直接略过，下面说下logstash是如何完成日志收集工作的。

Logstash日志收集可以分成3大块，input, filter和output, 可以通过启动时指定配置文件来配置。

```bash
./bin/logstash -f config_file 
```

###Input是配置日志源的，这里使用文件做例子:

```config
input {
  file {
    #日志存储格式 /web/mywebsite/logs/201407/login_errr_10.log
    path => "/web/mywebsite/logs/*/*"  
    type => "web_log"
  }

  file {
    #日志存储格式 /web/mywebsite/logs/apache_access.log
    path => "/web/mywebsite/logs/*"
    type => "access_log"
  }
}
```

可以看到指定输入的时候可以使用通配符监视多个文件。

###Filter用来对日志做过滤，比较常用的filter有

[grok](http://logstash.net/docs/1.4.2/filters/grok) 通过正则表达式把每一行的日志做匹配，把非结构化的信息变成结构化的
[date](http://logstash.net/docs/1.4.2/filters/date) 可以用指定字段的值作为当前log的时间戳，而不是使用导入log时的系统时间。

解释起来比较拗口，看实际的配置例子更容易理解。

```config
filter {
  #可以使用条件语句的, 中括号里面的[type]代表引用type这个字段的值
  if [type] == "access_log" {
    grok {
      #COMBINEDAPACHELOG, 预定义的一组正则
      match => { "message" => "%{COMBINEDAPACHELOG}" }
    }

    #这里把[timestamp]这个字段的值按照"dd/MMM/yyyy:HH:mm:ss Z"的格式解析成日期，作为本条log的产生日期。
    date {
      match => [ "timestamp" , "dd/MMM/yyyy:HH:mm:ss Z" ]
    }
  } else {

    #匹配这样的消息:  2014-07-14T22:10:51-07:00 INFO (6): log body
    grok {
      match => {"message"  => "%{TIMESTAMP_ISO8601:timestamp} %{DATA:data}"}
    }

    #这里把[timestamp]这个字段的值按照iso8601的格式解析
    date {
      match => [ "timestamp" , "ISO8601" ]
    }
  }
}
```
上面的COMBINEDAPACHELOG是一组预定义的正则，[https://github.com/logstash/logstash/tree/v1.4.2/patterns](https://github.com/logstash/logstash/tree/v1.4.2/patterns)这里能看到更多的。

使用上面配置解析出的log如下

```config
{
	"message" => "2014-07-14T22:12:26-07:00 INFO (6): Log detail",
	"@version" => "1",
	"@timestamp" => "2014-07-15T05:12:26.000Z",
	"type" => "web_log",
	"host" => "myserver.local",
	"path" => "/web/mywebsite/logs/201407/login_error_14.txt",
	"timestamp" => "2014-07-14T22:12:26-07:00"
}
```
##在使用Filter转换好格式后，就可以配置output输出了

```config
output {
  stdout { codec => rubydebug }
  elasticsearch_http { host => localhost port => 9200}
}
```
可以看到，我配置了2个输出，一个使用rubydebug的格式打印到console上，一个使用elasticsearch_http输出到ES。

以上就是logstash的配置过程，最重点的部分在filter上。数据到了ES之后，使用kibana查询可以参考这里的[介绍](http://www.elasticsearch.org/guide/en/kibana/current/)
