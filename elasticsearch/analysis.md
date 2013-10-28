analysis

基本概念
===

全文搜索引擎会用某种算法对要建索引的文档进行分析， 从文档中提取出若干Token(词元)， 这些算法称为Tokenizer(分词器)， 这些Token会被进一步处理， 比如转成小写等， 这些处理算法被称为Token Filter(词元处理器), 被处理后的结果被称为Term(词)， 文档中包含了几个这样的Term被称为Frequency(词频)。 引擎会建立Term和原文档的Inverted Index(倒排索引)， 这样就能根据Term很快到找到源文档了。    文本被Tokenizer处理前可能要做一些预处理， 比如去掉里面的HTML标记， 这些处理的算法被称为Character Filter(字符过滤器)， 这整个的分析算法被称为Analyzer(分析器)。 

ES内置了很多Analyzer, 还有很多第三方的Analyzer插件， 比如一些处理中文的Analyzer(中文分词)。 

analyzer、 tokenizer、 filter可以在elasticsearch.yml 配置， 下面是配置例子

```json
index :
    analysis :
        analyzer :
            standard :
                type : standard
                stopwords : [stop1, stop2]
            myAnalyzer1 :
                type : standard
                stopwords : [stop1, stop2, stop3]
                max_token_length : 500
            # configure a custom analyzer which is
            # exactly like the default standard analyzer
            myAnalyzer2 :
                tokenizer : standard
                filter : [standard, lowercase, stop]
        tokenizer :
            myTokenizer1 :
                type : standard
                max_token_length : 900
            myTokenizer2 :
                type : keyword
                buffer_size : 512
        filter :
            myTokenFilter1 :
                type : stop
                stopwords : [stop1, stop2, stop3, stop4]
            myTokenFilter2 :
                type : length
                min : 0
                max : 2000

```

analyzer
===

ES内置若干analyzer, 另外还可以用内置的character filter, tokenizer, token filter组装一个analyzer(custom analyzer)， 比如

```json
index :
    analysis :
        analyzer :
            myAnalyzer :
                tokenizer : standard
                filter : [standard, lowercase, stop]
```

如果你要使用第三方的analyzer插件，需要先在配置文件elasticsearch.yml中注册, 下面是配置IkAnalyzer的例子

```json
index:
  analysis:
    analyzer:      
      ik:
          alias: [ik_analyzer]
          type: org.elasticsearch.index.analysis.IkAnalyzerProvider

```

当一个analyzer在配置文件中被注册到一个名字(logical name)下后，在mapping定义或者一些API里就可以用这个名字来引用该analyzer了，比如

```json
"message": {
    "type": "string",
    "indexAnalyzer": "ik",
    "searchAnalyzer": "ik"
}
```

如果没有指定索引和搜索用的analyzer，ES会用默认的analyzer来处理，也就是名字(logical name)为`default`, `default_index`, `default_search`的analyzer。 
从名字可以看出来，`default`是索引和搜索时用的默认的analyzer，`default_index`是索引时用的默认的analyzer， 
`default_search`是查询时用的默认analyzer。

下面是在elasticsearch.yml中配置默认analyzer的例子

```json
index:
  analysis:
    analyzer:
        default_index:
            tokenizer: standard
            filter: [standard, lowercase, my_synonym, my_snow]
        default_search:
            tokenizer: standard
            filter: [standard, lowercase, stop]
```
或者用这种格式

```json
index.analysis.analyzer.default.type : "mmseg"

```

一个analyzer可以起若干别名，比如在下面的例子中，standard analyzer可以用alias1或者alias2来引用

```json
index :
  analysis :
    analyzer。 :
      standard :
        alias: [alias1, alias2]
        type : standard
        stopwords : [test1, test2, test3]
```

下面是内置的一些analyzer。

| analyzer              | logical name  | description                               |
| ----------------------|:-------------:| :-----------------------------------------|
| standard analyzer     | standard      | standard tokenizer, standard filter, lower case filter, stop filter |
| simple analyzer       | simple        | lower case tokenizer                      |
| stop analyzer         | stop          | lower case tokenizer, stop filter         |
| keyword analyzer      | keyword       | 不分词，内容整体作为一个token(not_analyzed) |
| pattern analyzer      | whitespace    | 正则表达式分词，默认匹配\W+                 |
| language analyzers    | [lang](http://www.elasticsearch.org/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html)  | 各种语言 |
| snowball analyzer     | snowball      | standard tokenizer, standard filter, lower case filter, stop filter, snowball filter |
| custom analyzer       | custom        | 一个Tokenizer, 零个或多个Token Filter, 零个或多个Char Filter |

tokenizer
===

ES内置的tokenizer列表。

| tokenizer             | logical name  | description                           |
| ----------------------|:-------------:| :-------------------------------------|
| standard tokenizer    | standard      |                                       |
| edge ngram tokenizer  | edgeNGram     |                                       |
| keyword tokenizer     | keyword       | 不分词                                 |
| letter analyzer       | letter        | 按单词分                               |
| lowercase analyzer    | lowercase     | letter tokenizer, lower case filter   |
| ngram analyzers       | nGram         |                                       |
| whitespace analyzer   | whitespace    | 以空格为分隔符拆分                      |
| pattern analyzer      | pattern       | 定义分隔符的正则表达式                  |
| uax email url analyzer| uax_url_email | 不拆分url和email                       |
| path hierarchy analyzer| path_hierarchy| 处理类似`/path/to/somthing`样式的字符串|


token filter
===

ES内置的token filter列表。

| token filter          | logical name  | description                           |
| ----------------------|:-------------:| :-------------------------------------|
| standard filter       | standard      |                                       |
| ascii folding filter  | asciifolding  |                                       |
| length filter         | length        | 去掉太长或者太短的                      |
| lowercase filter      | lowercase     | 转成小写                               |
| ngram filter          | nGram         |                                       |
| edge ngram filter     | edgeNGram     |                                       |
| porter stem filter    | porterStem    | 波特词干算法                            |
| shingle filter        | shingle       | 定义分隔符的正则表达式                  |
| stop filter           | stop          | 移除 stop words                        |
| word delimiter filter | word_delimiter| 将一个单词再拆成子分词                   |
| stemmer token filter  | stemmer       |                                        |
| stemmer override filter| stemmer_override|                                     |
| keyword marker filter | keyword_marker|                                        |
| keyword repeat filter | keyword_repeat|                                        |
| kstem filter          | kstem         |                                        |
| snowball filter       | snowball      |                                        |
| phonetic filter       | phonetic      | [插件](https://github.com/elasticsearch/elasticsearch-analysis-phonetic) |
| synonym filter        | synonyms      | 处理同义词                              |
| compound word filter  | dictionary_decompounder, hyphenation_decompounder | 分解复合词  |
| reverse filter        | reverse       | 反转字符串                              |
| elision filter        | elision       | 去掉缩略语                              |
| truncate filter       | truncate      | 截断字符串                              |
| unique filter         | unique        |                                        |
| pattern capture filter| pattern_capture|                                       |
| pattern replace filte | pattern_replace| 用正则表达式替换                        |
| trim filter           | trim          | 去掉空格                                |
| limit token count filter| limit       | 限制token数量                           |
| hunspell filter       | hunspell      | 拼写检查                                |
| common grams filter   | common_grams  |                                        |
| normalization filter  | arabic_normalization, persian_normalization |          |


character filter
===

ES内置的character filter列表

| character filter          | logical name  | description               |
| --------------------------|:-------------:| :-------------------------|
| mapping char filter       | mapping       | 根据配置的映射关系替换字符   |
| html strip char filter    | html_strip    | 去掉HTML元素               |
| pattern replace char filter| pattern_replace| 用正则表达式处理字符串    |


icu plugin
===

[icu analysis 插件](https://github.com/elasticsearch/elasticsearch-analysis-icu)

