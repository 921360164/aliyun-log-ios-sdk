# log service ios producer

## 功能特点

* 异步
    * 异步写入，客户端线程无阻塞
* 聚合&压缩 上传
    * 支持按超时时间、日志数、日志size聚合数据发送
    * 支持lz4压缩
* 多客户端
	* 可同时配置多个客户端，每个客户端可独立配置采集优先级、缓存上限、目的project/logstore、聚合参数等 (多客户端，断点续传文件路径不能相同) 
* 缓存
    * 支持缓存上限可设置
    * 超过上限后日志写入失败
* 自定义标识
    * 支持设置自定义tag、topic
* 断点续传功能
    * 每次发送前会把日志保存到本地的binlog文件，只有发送成功才会删除，保证日志上传At Least Once

![image.png](https://test-lichao.oss-cn-hangzhou.aliyuncs.com/pic/099B6EC1-7305-4C18-A1CF-BA2CCD1FBDBC.png)

## 性能测试

* 开启断点续传

| 发送 条/每秒 | cpu占用 |  内存占用(MB) | 上传速(MB/min) |
| --- | --- | --- | --- |
| 1 | <1% | 14.2 | 0.046 |
| 10 | 1% | 14.5 | 0.435 |
| 100 | 6% | 14.5 | 4.31 |
| 300 | 11% | 14.7 | 12.80 |

* 不开启断点续

| 发送 条/每秒 | cpu占用 |  内存占用(MB) | 上传速(MB/min) |
| --- | --- | --- | --- |
| 1 | <1% | 14 | 0.046 |
| 10 | 1% | 14 | 0.437 |
| 100 | 6% | 14.1 | 4.50 |
| 300 | 9% | 14.4 | 13.47 |

## Podfile
```
pod 'AliyunLogProducer', '~> 2.1.0'
```

## swift 配置说明

### import

```
import AliyunLogProducer
```

### 创建config

https://help.aliyun.com/document_detail/29064.html

```
let endpoint = "project's_endpoint";
let project = "project_name";
let logstore = "logstore_name";
let accesskeyid = "your_accesskey_id";
let accesskeysecret = "your_accesskey_secret";

let config = LogProducerConfig(endpoint:endpoint, project:project, logstore:logstore, accessKeyID:accesskeyid, accessKeySecret:accesskeysecret)!
// 指定sts token 创建config，过期之前调用ResetSecurityToken重置token
// let config = LogProducerConfig(endpoint:endpoint, project:project, logstore:logstore, accessKeyID:accesskeyid, accessKeySecret:accesskeysecret, securityToken:securityToken)
```

### 配置config & 创建client

```
        // 设置主题
        config.setTopic("test_topic")
        // 设置tag信息，此tag会附加在每条日志上
        config.addTag("test", value:"test_tag")
        // 每个缓存的日志包的大小上限，取值为1~5242880，单位为字节。默认为1024 * 1024
        config.setPacketLogBytes(1024*1024)
        // 每个缓存的日志包中包含日志数量的最大值，取值为1~4096，默认为1024
        config.setPacketLogCount(1024)
        // 被缓存日志的发送超时时间，如果缓存超时，则会被立即发送，单位为毫秒，默认为3000
        config.setPacketTimeout(3000)
        // 单个Producer Client实例可以使用的内存的上限，超出缓存时add_log接口会立即返回失败
        // 默认为64 * 1024 * 1024
        config.setMaxBufferLimit(64*1024*1024)
        // 发送线程数，默认为1
        config.setSendThreadCount(1)
        
        // 1 开启断点续传功能， 0 关闭
        // 每次发送前会把日志保存到本地的binlog文件，只有发送成功才会删除，保证日志上传At Least Once
        config.setPersistent(1)
        // 持久化的文件名，需要保证文件所在的文件夹已创建。
        let file = NSSearchPathForDirectoriesInDomains(FileManager.SearchPathDirectory.documentDirectory, FileManager.SearchPathDomainMask.userDomainMask, true).first
        let path = file! + "/log.dat"
        config.setPersistentFilePath(path)
        // 是否每次AddLog强制刷新，高可靠性场景建议打开
        config.setPersistentForceFlush(1)
        // 持久化文件滚动个数，建议设置成10。
        config.setPersistentMaxFileCount(10)
        // 每个持久化文件的大小，建议设置成1-10M
        config.setPersistentMaxFileSize(1024*1024)
        // 本地最多缓存的日志数，不建议超过1M，通常设置为65536即可
        config.setPersistentMaxLogCount(65536)
        let callbackFunc: on_log_producer_send_done_function = {config_name,result,log_bytes,compressed_bytes,req_id,error_message,raw_buffer,user_param in
            let res = LogProducerResult(rawValue: Int(result))
//            print(res!)
//            let req = String(cString: req_id!)
//            print(req)
//            print(log_bytes)
//            print(compressed_bytes)
        }
        client = LogProducerClient(logProducerConfig:config, callback:callbackFunc)
//        client = LogProducerClient(logProducerConfig:config)
```

### 写数据

```
let log = Log()
log.putContent("k1", value:"v1")
log.putContent("k2", value:"v2")

// addLog第二个参数flush，是否立即发送，1代表立即发送，不设置时默认为0
let res = client?.add(log, flush:0)
```

## oc 配置说明

### import

```
#import "AliyunLogProducer/AliyunLogProducer.h"
```

### 创建config

https://help.aliyun.com/document_detail/29064.html

```
NSString* endpoint = @"project's_endpoint";
NSString* project = @"project_name";
NSString* logstore = @"logstore_name";
NSString* accesskeyid = @"your_accesskey_id";
NSString* accesskeysecret = @"your_accesskey_secret";

LogProducerConfig* config = [[LogProducerConfig alloc] initWithEndpoint:endpoint project:project logstore:logstore accessKeyID:accesskeyid accessKeySecret:accesskeysecret];
// 指定sts token 创建config，过期之前调用ResetSecurityToken重置token
// LogProducerConfig* config = [[LogProducerConfig alloc] initWithEndpoint:endpoint project:project logstore:logstore accessKeyID:accesskeyid accessKeySecret:accesskeysecret securityToken:securityToken];
```

### 配置config & 创建client

```
// 设置主题
[config SetTopic:@"test_topic"];
// 设置tag信息，此tag会附加在每条日志上
[config AddTag:@"test" value:@"test_tag"];
// 每个缓存的日志包的大小上限，取值为1~5242880，单位为字节。默认为1024 * 1024
[config SetPacketLogBytes:1024*1024];
// 每个缓存的日志包中包含日志数量的最大值，取值为1~4096，默认为1024
[config SetPacketLogCount:1024];
// 被缓存日志的发送超时时间，如果缓存超时，则会被立即发送，单位为毫秒，默认为3000
[config SetPacketTimeout:3000];
// 单个Producer Client实例可以使用的内存的上限，超出缓存时add_log接口会立即返回失败
// 默认为64 * 1024 * 1024
[config SetMaxBufferLimit:64*1024*1024];
// 发送线程数，默认为1
[config SetSendThreadCount:1];

// 1 开启断点续传功能， 0 关闭
// 每次发送前会把日志保存到本地的binlog文件，只有发送成功才会删除，保证日志上传At Least Once
[config SetPersistent:1];
// 持久化的文件名，需要保证文件所在的文件夹已创建。
[config SetPersistentFilePath:Path];
// 是否每次AddLog强制刷新，高可靠性场景建议打开
[config SetPersistentForceFlush:1];
// 持久化文件滚动个数，建议设置成10。
[config SetPersistentMaxFileCount:10];
// 每个持久化文件的大小，建议设置成1-10M
[config SetPersistentMaxFileSize:1024*1024];
// 本地最多缓存的日志数，不建议超过1M，通常设置为65536即可
[config SetPersistentMaxLogCount:65536];

//创建client
client = [[LogProducerClient alloc] initWithLogProducerConfig:config];
```

### 写数据

```
Log* log = [[Log alloc] init];
[log PutContent:@"k1" value:@"v1"];
[log PutContent:@"k2" value:@"v2"];

// addLog第二个参数flush，是否立即发送，1代表立即发送，不设置时默认为0
LogProducerResult* res = [client AddLog:log flush:0];
```

