# HAO ES 分词器
## 简介
> **如何开发一个ES分词插件**请参考 [这里](https://github.com/tenlee2012/elasticsearch-analysis-demo)

主要参考了 [IK](https://github.com/medcl/elasticsearch-analysis-ik) 和 [HanLP](https://github.com/hankcs/HanLP)
其中有的源码是直接搬运的。
相比IK，比IK更智能，更准确，更快。
相比HanLp，比HanLP更轻量，分词更可控，没有一些智能的预测功能，并且HanLP也没有官方的ES插件。


提供 
Analyzer: `hao_search_mode`, `hao_index_mode`
Tokenizer: `hao_search_mode`, `hao_index_mode`

Versions
--------

Git tag | ES version
-----------|-----------
master | ES最新稳定版
v7.7.1 | 7.7.1
vX.Y.Z | X.Y.Z

## 使用
### 安装
方式1. `bin/elasticsearch-plugin install file:///Users/xiaoming/Download/analysis-hao.zip`

方式2. 解压后，放在es plugins目录即可。

最后重启ES

### ES 版本升级
如果没有你需要的对应ES版本，修改`pom.xml`->`elasticsearch.version`的值为对应版本，然后执行
`mvn clean package -Dmaven.test.skip=true`，就可以得到插件的`zip`安装包。

### 自定义分词器
下面是自定义分词器可用的配置项

---
配置项参数 | 功能 | 默认值 
----|---|---
`enableIndexMode` | 是否使用index模式，index模式为细颗粒度。| `hao_search_mode`为`false`，`hao_index_mode`为`true`,细颗粒度适合Term Query,粗颗粒度适合Phrase查询
`enableFallBack` | 如果分词报错，是否启动最细粒度分词，即按字分。建议`search_mode`使用，不至于影响用户搜索。`index_mode`不启动，以便及时报错告警通知。| `false`不启动降级
`enableFailDingMsg` | 是否启动失败钉钉通知,通知地址为`HttpAnalyzer.cfg.xml`的`dingWebHookUrl`字段。| `false`
`enableSingleWord` | 是否使用细粒度返回的单字。比如`体力值`，分词结果只存`体力值`,`体力`,而不存`值` | `false`

### HaoAnalyzer.cfg.xml 配置

---
参数| 功能 | 备注
--- | --- | ---
`baseDictionary` |基础词库文件名 | 放在插件`config`目录或者es的`config`目录，不用更改 
`customerDictionaryFile` | 用户自定义远程词库文件| 会存储在插件`config`目录或者es的`config`目录
`remoteFreqDict` | 远程用户自定义词库文件 | 方便热更新，热更新通过下面两个参数定时更新。 
`syncDicTim` | 远程词库第一次同步时间 `hh:mm:ss` | - 
`syncDicPeriodTime` | 远程词库同步时间间隔,秒 | 比如 `syncDicTime=20:00:00,syncDicPeriodTime=86400`，则是每天20点同步
`dingWebHookUrl` | 钉钉机器人url | 用于分词异常，同步词库异常/成功通知|
`dingMsgContent` | 机器人通知文案 | 注意配置钉钉机器人的时候关键词要和这个文案匹配，不然会消息发送失败

### 词库说明
> 优先读取 `{ES_HOME}/config/analysis-hao/`目录，没有读取 `{ES_HOME}/plugins/analysis-hao/config`目录下的文件

 - 基础词库
基础词库是`base_dictionary.json`,是一个json文件，`key`为词，`value`为词频（`int`)。是可以修改的，可以添加词，可以修改词频。
例如：`奋发图强` 分词结果是 `奋`, `发图`, `强`, 是因为`发图`这个词的词频太高了（因为出现次数高），则可以降低词频，手动修改`base_dictionary.json`文件就好了。
 - 远程词库
用户自定义词库会按照配置的时间和周期定期执行。
从远程词库更新完成后会自动覆盖现在的`customerDictionaryFile`。
远程词库的文件格式**每行**格式为 `{词},{词频},{是否元词}`, 例如`俄罗斯,1000,1`。
是否元词字段解释：
`1`代表是元词，不会再细拆分，`俄罗斯`不会再拆分成`俄`和`罗斯`（罗斯是常用人名）。这样搜`罗斯`就不会把`俄罗斯`相关文档召回。
`0`就是可以继续细拆分，比如`奋发图强`



### 示例索引demo
建索引：
```
PUT test/
{
  "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "search_analyzer": {
            "filter": [
              "lowercase"
            ],
            "char_filter": [
              "html_strip"
            ],
            "type": "custom",
            "tokenizer": "my_search_token"
          },
          "title_analyzer": {
            "filter": [
              "lowercase"
            ],
            "char_filter": [
              "html_strip"
            ],
            "type": "custom",
            "tokenizer": "my_title_index_token"
          }
        },
        "tokenizer": {
          "my_title_index_token": {
            "enableOOV": "false",
            "enableFailDingMsg": "true",
            "type": "hao_index_mode",
            "enableSingleWord": "true",
            "enableFallBack": "true"
          },
          "my_search_token": {
            "enableOOV": "false",
            "enableFailDingMsg": "true",
            "type": "hao_search_mode",
            "enableSingleWord": "true",
            "enableFallBack": "true"
          }
        }
      },
      "number_of_replicas": "0"
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "index_options": "offsets",
        "analyzer": "title_analyzer",
        "search_analyzer": "search_analyzer"
      }
    }
  }
}
```
测试分词
```
test/_analyze
{
  "analyzer": "title_analyzer",
  "text": "奋发图强打篮球有利于提高人民生活，有的放矢，中华人民共和国家庭宣传委员会宣。🐶"
}

test/_analyze
{
  "analyzer": "search_analyzer",
  "text": "奋发图强打篮球有利于提高人民生活，有的放矢，中华人民共和国家庭宣传委员会宣。🐶"
}
```

