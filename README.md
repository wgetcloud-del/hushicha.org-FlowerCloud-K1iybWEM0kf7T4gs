
做全文搜索，es比较好用，安装可能有点费时费力。mysql安装就不说了。主要是elastic8\.4\.0\+kibana8\.4\.0\+logstash\-8\.16\.1，可视化操作及少量netcore查询代码。


安装elastic8\.4\.0\+kibana8\.4\.0使用docker\-desktop，logstash\-8\.16\.1是线程解压执行文件。


* **1\.**docker\-compose.yml 如下: 首先使用docker network创建一个es\-net内部通讯网络,这样kibana连接es可以通过容器名ELASTICSEARCH\_HOSTS\=http://elasticsearch:9200，此作为单机测试使用单机的es.




```
services:

  elasticsearch:
    container_name: elasticsearch
    image: docker.elastic.co/elasticsearch/elasticsearch:8.4.0
    environment:
      - discovery.type=single-node
    ulimits:
      memlock:
        soft: -1
        hard: -1
    cap_add:
      - IPC_LOCK
    ports:
      - "9200:9200"
    networks:
      - es-net 

  kibana:
    container_name: kibana
    image: docker.elastic.co/kibana/kibana:8.4.0
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    networks:
      - es-net

networks:
  es-net:
    driver: bridge
```


作为es的8以上版本是有账号密码和crt证书的，需要做如下配置：


安装好es后默认给一个elastic账号,需要重置一下密码,进入es容器执行重置密码命令，会给你一个密码。




```
docker exec  -it -u root elasticsearch /bin/bash
bin/elasticsearch-reset-password -u elastic
```


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204123646088-1706594322.png)


 


这里登录的其实是https带证书的，但是kibana使用的是http的，所以在容器内部，config/elasticsearch.yml中需要把下面的两个参数置为false ，生产环境不建议这么操作。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204120214927-1097291483.png)


 因为es带账号密码，所以kibana连接es也需要账号密码信息，但是默认的elastic是超级管理员，kibana默认是不支持的，需要自己新建账号。但是es默认是给了账号的，用他的就行。自己新建es账号给一个超级管理员角色依然没有重建所应权限，导致kibana起不来，用kibana\_system就行。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204120644276-1179890360.png)


进入es容器内部给kibana\_system重置一个密码，用下面的命令在内部调用也行，我设置的elastic和kibana\_system的密码一样，方便使用。




```
curl -u elastic:DiVnR2F6OGYmP+Ms+n2o -X POST "http://localhost:9200/_security/user/kibana_system/_password" -H 'Content-Type: application/json' -d'
{
  "password": "DiVnR2F6OGYmP+Ms+n2o"
}
'
```


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204120938900-526591861.png)


* **2\.** 然后在kibana容器中，加上账号密码信息即可，重启。还有最后一行加上i18n.locale: zh\-CN  ,改变ui为中文。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204121235759-1473953964.png)


 然后通过开发工具就可以做es的调试了,这里注意下需要中文分词的可以去 https://github.com/infinilabs/analysis\-ik/releases 下载对应版本8\.4\.0的中文分词器 ，改个名放到es容器内plugins中去。也可以自定义分词文件丢进去


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204122912305-1939149091.png)


 


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204121409895-271218147.png)


* 3\. 下面就是logstash安装跟mysql的同步了，测试数据如下：


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204123939694-874271362.png)


 


 首先去logstash官网下载对应的包，我选的版本是8\.16\.1，目录如下是可以通过控制台执行的。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204121714655-1832980795.png)


 这里只需要配置好mysql\-connector的驱动和链接信息即可。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204124102106-186464079.png)


 


 jdbc.conf文件内容如下:




```
input {
    stdin {}
    jdbc {
        type => "jdbc"
         # 数据库连接地址
        jdbc_connection_string => "jdbc:mysql://192.168.200.2:3306/bbs?characterEncoding=UTF-8&autoReconnect=true"
         # 数据库连接账号密码；
        jdbc_user => "admin"
        jdbc_password => "这是密码"
         # MySQL依赖包路径；
        jdbc_driver_library => "D:\software\logstash-8.16.1\mysql\mysql-connector-j-8.0.32.jar"
         # the name of the driver class for mysql
        jdbc_driver_class => "com.mysql.jdbc.Driver"
         # 数据库重连尝试次数
        connection_retry_attempts => "3"
         # 判断数据库连接是否可用，默认false不开启
        jdbc_validate_connection => "true"
         # 数据库连接可用校验超时时间，默认3600S
        jdbc_validation_timeout => "3600"
         # 开启分页查询（默认false不开启）；
        jdbc_paging_enabled => "true"
         # 单次分页查询条数（默认100000,若字段较多且更新频率较高，建议调低此值）；
        jdbc_page_size => "500"
         # statement为查询数据sql，如果sql较复杂，建议配通过statement_filepath配置sql文件的存放路径；
         # sql_last_value为内置的变量，存放上次查询结果中最后一条数据tracking_column的值，此处即为ModifyTime；
         # statement_filepath => "mysql/jdbc.sql"
        statement => "SELECT    ArticleID,UserID,ArticleTitle,ArticleContent,ImageAddress,StandPoint,PublishTime,`Status`,Likes,    Shares,Comments,Reports,    Sort,PublishingMode,SourceType,Reply,IsTop,TopEndTime,Hot,EditUserId,CreatedTime,EditTime,UserType,UserNickname,ForbiddenState,PublishDateTime,TopArea,SubscribeType,CollectionCount,Articletype,NewsID,CommentUserCount,TopStartTime,`View`,ViewDuration,Forwardings,ForwardingFId,Freshness,Shelf_Reason,AuditTime FROM bbs_articles" 
         # 是否将字段名转换为小写，默认true（如果有数据序列化、反序列化需求，建议改为false）；
        lowercase_column_names => false
         # Value can be any of: fatal,error,warn,info,debug，默认info；
        sql_log_level => warn
         #
         # 是否记录上次执行结果，true表示会将上次执行结果的tracking_column字段的值保存到last_run_metadata_path指定的文件中；
        record_last_run => true
         # 需要记录查询结果某字段的值时，此字段为true，否则默认tracking_column为timestamp的值；
        use_column_value => true
         # 需要记录的字段，用于增量同步，需是数据库字段
        tracking_column => "PublishTime"
         # Value can be any of: numeric,timestamp，Default value is "numeric"
        tracking_column_type => timestamp
         # record_last_run上次数据存放位置；
        last_run_metadata_path => "mysql/last_id.txt"
         # 是否清除last_run_metadata_path的记录，需要增量同步时此字段必须为false；
        clean_run => false
         #
         # 同步频率(分 时 天 月 年)，默认每分钟同步一次；
        schedule => "* * * * *"
    }
}

filter {
    json {
        source => "message"
        remove_field => ["message"]
    }
    # convert 字段类型转换，将字段TotalMoney数据类型改为float；
    mutate {
        convert => {
            #    "TotalMoney" => "float"
        }
    }
}
output {
    elasticsearch {
         # host => "127.0.0.1"
         # port => "9200"
         # 配置ES集群地址
         # hosts => ["192.168.1.1:9200", "192.168.1.2:9200", "192.168.1.3:9200"]
         hosts => ["127.0.0.1:9200"]
            user => "elastic"
            password => "DiVnR2F6OGYmP+Ms+n2o"
            ssl => false
         # 索引名字，必须小写
        index => "bbs_act"
         # 数据唯一索引（建议使用数据库KeyID）
        document_id => "%{ArticleID}"
    }
    stdout {
        codec => json_lines
    }
}
```


配置文成后执行该命令，数据实时同步开始




```
bin\logstash.bat -f mysql\jdbc.conf
```


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204122019171-940798654.png)


 可以通过kibana的discover查看数据，也可以通过开发工具查询，elk日志就是这么玩。


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204122235396-1807036850.png)


 


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204122426285-1261590532.png)


* 4\. 下面就是代码，这里的实体没给全，注意实体需要给Text的Name属性，否则会解析不到数据的：




```
 public class ArticleEsContext : EsBase
 {
     public ArticleEsContext(EsConfig esConfig) : base(esConfig)
     {
     }

     public override string IndexName => "bbs_act";

     public async Task> GetArticles(ArticleParameter parameter)
     {
         var client = _esConfig.GetClient(IndexName);

         // 计算分页的起始位置
         var from = (parameter.PageNumber - 1) * parameter.PageSize;

         var searchResponse = await client.SearchAsync(s => s
             .Index(IndexName)
             .Query(q => q
                 .Bool(b => b
                     .Should(
                         sh => sh.Match(m => m
                             .Field(f => f.ArticleTitle)  // 查询 ArticleTitle
                             .Query(parameter.KeyWords)
                             .Fuzziness(Fuzziness.Auto)  // 启用模糊查询
                         ),
                         sh => sh.Match(m => m
                             .Field(f => f.ArticleContent)  // 查询 ArticleContent
                             .Query(parameter.KeyWords)
                             .Fuzziness(Fuzziness.Auto)  // 启用模糊查询
                         )
                     )
                     .MinimumShouldMatch(1)  // 至少一个条件必须匹配
                 )
             )
             .From(from)  // 设置分页的起始位置
             .Size(parameter.PageSize)  // 设置每页大小
         );

         if (!searchResponse.IsValid)
         {
             Console.WriteLine(searchResponse.DebugInformation);
             return new List();
         }

         return searchResponse.Documents.ToList();
     }
 }

 public class ArticleDto
 {
     [Text(Name = "ArticleID")]
     public int ArticleId { get; set; }
     [Text(Name = "ArticleTitle")]
     public string ArticleTitle { get; set; }
     [Text(Name = "ArticleContent")]
     public string ArticleContent { get; set; }
     [Date(Name = "CreatedTime")]
     public DateTime CreatedTime { get; set; }
 }
```


代码调用结果如下：


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204123332809-1486568606.png)


 


![](https://img2024.cnblogs.com/blog/1099890/202412/1099890-20241204123341805-685365552.png)


 



```
d5a34f86-c01f-42e1-a93f-a0aabe2e1246
```

 本博客参考[FlowerCloud机场](https://yunbeijia.com)。转载请注明出处！
