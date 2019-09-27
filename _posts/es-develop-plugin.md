---
title: elasticsearch解析器插件开发介绍
categories:
- elasticsearch
tags:
- elasticsearch,es插件
---

# 概要
本文主要介绍如何开发ES解析器插件。开发的解析插件的核心功能是可以将词"abcd"分为\["abcd","bcd","cd","d"\]。源码地址[在此](https://github.com/lxl910128/analysis-rockstone-plugin)。

在开始前还需要说明的是，根据高人指点，上篇[文章](http://blog.gaiaproject.club/es-contains-search/)中提四种分词方案即将"abcd"分词为\["abcd","bcd","cd","d"\]，其实使用ES自带的`edge_ngram` tokenizer加`reverse` token filter就可实现相似的功能。即将"abcd"分词为\["a","ba","cba","dcba"\]，搜索时借助`prefix`查询就可以完成字符串包涵的搜索需求。这里还有一个需要注意的点是，使用`prefix`查询时需要自行将查询词逆序。当然也可以使用`match_phrase_prefix`使用`analyzer`配置只带`reverse` token filter的解析器对查询词进行反转。使用`match_phrase_prefix`还有个好出会自行根据出现频率排序。

# 预备知识点
1. 在es中一串字符串如果要被索引，需要经过对应的解析器`Analyzers`将其转化出`terms`和`tokens`。
2. `term`是搜索的最小单元
3. `token`是一种在对文本进行分词时产生的对象，它包括term，term在文本中的位置，term起止位置等信息。
4. es解析器在分词时有3个步骤。`character filters`、`tokenizer`和`token filters`
5. `character filters`将原始文本作为字符流接收，可以对字符流进行增删改，最后将新的流输出。比如无差别将大写字母变小写并删除符号：'HELLO WORD! LOL' -> 'hello word lol'
6. `tokenizer`接收`character filters`转化后的字符流然后将它切分成词输出`token`流。对于英文一般采用空格切分'hello word' -> \['hello','word','lol'\]
7. `token filter`接收`token`流，并增删改tokens。\['hello','word','lol'\] -> \['hello','word','smile'\]

举个例子，使用standard Analyzer对"The 2 QUICK Brown-Foxes jumped over the lazy dog's bone."进行解析时，首先会用Standard Tokenizer切分句子(该analyzer默认没有Character Filters)，然后使用(Lower Case Token Filter)将所有字符转化为小写。最后会生成如下这些Token。
```json
{
    "tokens": [
        {
            "token": "the",
            "start_offset": 0, //开始位置
            "end_offset": 3,  //结束位置
            "type": "<ALPHANUM>", //词类型
            "position": 0 //偏移
        },
        {
            "token": "2",
            "start_offset": 4,
            "end_offset": 5,
            "type": "<NUM>",
            "position": 1
        },
        {
            "token": "quick",
            "start_offset": 6,
            "end_offset": 11,
            "type": "<ALPHANUM>",
            "position": 2
        },
        {
            "token": "brown",
            "start_offset": 12,
            "end_offset": 17,
            "type": "<ALPHANUM>",
            "position": 3
        },
        {
            "token": "foxes",
            "start_offset": 18,
            "end_offset": 23,
            "type": "<ALPHANUM>",
            "position": 4
        },
        {
            "token": "jumped",
            "start_offset": 24,
            "end_offset": 30,
            "type": "<ALPHANUM>",
            "position": 5
        },
        {
            "token": "over",
            "start_offset": 31,
            "end_offset": 35,
            "type": "<ALPHANUM>",
            "position": 6
        },
        {
            "token": "the",
            "start_offset": 36,
            "end_offset": 39,
            "type": "<ALPHANUM>",
            "position": 7
        },
        {
            "token": "lazy",
            "start_offset": 40,
            "end_offset": 44,
            "type": "<ALPHANUM>",
            "position": 8
        },
        {
            "token": "dog's",
            "start_offset": 45,
            "end_offset": 50,
            "type": "<ALPHANUM>",
            "position": 9
        },
        {
            "token": "bone",
            "start_offset": 51,
            "end_offset": 55,
            "type": "<ALPHANUM>",
            "position": 10
        }
    ]
}
```

这些tokens包括主要包含以下这些Terms: [the, 2, quick, brown, foxes, jumped, over, the, lazy, dog's, bone ]

# 插件介绍
ES插件主要是用来自定义增强ES核心功能的。主要可以扩展的功能包括：
* `ScriptPlugin`脚本插件，ES默认的脚本语言是Painless，可自定义其它脚本语言，java、js等。
* `AnalysisPlugin`解析器插件，可以扩展character filters、tokenizer和token filter。
* `DiscoveryPlugin`发现插件，使集群可以发现节点，如使建立在AWS上的集群可以发现节点。
* `ClusterPlugin`集群插件，增强对集群的管理，如控制分片位置。
* `IngestPlugin`摄取插件，增强节点的ingest功能，例如可以在ingest中通过tika解析ppt、pdf内容。
* `MapperPlugin`映射插件，可添加新的字段类型。
* `SearchPlugin`搜索插件，扩展搜索功能，如添加新的搜索类型，高亮类型等。
* `RepositoryPlugin`存储库插件，可添加新的存储库，如S3，Hadoop HDFS等。
* `ActionPlugin`API扩展插件，可扩展Http的API接口。
* `NetworkPlugin`网络插件，扩展ES底层的网络传输功能。
* `PersistentTaskPlugin`持续任务插件，用于注册持续任务的执行器。
* `EnginePlugin`实体插件，创建index时，每个enginePlugin会被运行，引擎插件可以检查索引设置以确定是否为给定索引提供engine factory。

# 插件开发
下面以我自行开发的插件为基础介绍插件开发主要流程。
## 官方教程
[官方教程](https://www.elastic.co/guide/en/elasticsearch/plugins/current/plugin-authors.html)对插件开发介绍的比较少。主要是告诉我们我们开发完成的插件应该以zip包的形式存在。在zip包的根目录种中最起码要包含我们开放的插件jar包以插件配置文件`plugin-descriptor.properties`。es是从配置文件认识自定义插件的。如果插件需要依赖其它jar包，则将其页放在zip根目录下即可。此次开发使用的说明文件如下。
```yaml
# Elasticsearch 插件说明文件,该文件必须命名为'plugin-descriptor.properties'并存放在插件根目录下
### java插件目录结构
#
# foo.zip <-- zip file for the plugin, with this structure:
#   <arbitrary name1>.jar <-- classes, resources, dependencies
#   <arbitrary nameN>.jar <-- any number of jars
#   plugin-descriptor.properties <-- example contents below:
#
#
### 插件必要的描述元素:
#
# 'description': 插件简述
description=${project.description}
#
# 'version': 插件版本
version=my first plugin. 
#
# 'name': 插件名称
name=analysis-rockstone
#
# 'classname': es需要加载类的全路径，该类需要继承Plugin类
classname=org.elasticsearch.plugin.analysis.rockstone.AnalysisRockstonePlugin
#
# 'java.version'
java.version=1.8
#
# 'elasticsearch.version' 插件适用的es版本,安装插件时会验证,需要严格匹配
elasticsearch.version=7.1.1
#
```

在插件配置文件`plugin-descriptor.properties`我们至少应该配置以下变量:
* **description**：插件介绍
* **version**：插件版本
* **name**：插件名称
* **classname**：要加载的插件类的全名。这个类需要继承`Plugin`类并实现插件类型接口，比如`ActionPlugin`、`AnalysisPlugin`等
* **java.version**：插件适用的java版本
* **elasticsearch.version**：插件适用的es版本


在插件成功编译成zip包后，我们可以适用`bin/elasticsearch-plugin install file:///path/to/your/plugin`命令来安装插件，或这将zip包直接放在es的`plugins/`目录下。

开发ES插件**总结**来说需要以下几步：
1. 需要编写一个继承`Plugin`实现插件类型的类。
2. 需要编写插件配置文件`plugin-descriptor.properties`。
3. 将这配置文件和jar包打成1个zip包。

## 项目初始化
根据官方教程可知，最终我们需要得到一个含有配置文件和jar包的zip包。这里我们借助`maven`及其`assembly`插件实现打包工作。pom.xml文件中关于`assembly`的配置如下
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <version>2.6</version>
            <configuration>
                <appendAssemblyId>false</appendAssemblyId>
                <!-- 将resources中的plugin-descriptor.properties放在根目录下 -->
                <outputDirectory>${project.build.directory}/releases/</outputDirectory>
                <descriptors><!--assembly使用的配置文件地址 -->
                    <descriptor>${basedir}/src/main/assemblies/plugin.xml</descriptor>
                </descriptors>
            </configuration>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>single</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
    </plugins>
</build>
```

`maven-assembly`使用的配置文件plugin.xml如下所示
```xml
<?xml version="1.0"?>
<assembly>
    <id>tokenizer-rockstone</id>
    <formats>   <!--打包方式-->
        <format>zip</format>
    </formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <fileSets>
        <fileSet> <!--配置要把什么文件打包到什么目录下-->
            <directory>${project.basedir}/config</directory>
            <outputDirectory>config</outputDirectory>
        </fileSet>
    </fileSets>
    <files>
        <file>
            <source>${project.basedir}/src/main/resources/plugin-descriptor.properties</source>
            <outputDirectory/>
            <filtered>true</filtered>
        </file>
    </files>
    <dependencySets><!--把相关的依赖包进行打包-->
        <dependencySet>
            <outputDirectory/>
            <useProjectArtifact>true</useProjectArtifact>
            <useTransitiveFiltering>true</useTransitiveFiltering>
            <excludes>
                <exclude>org.elasticsearch:elasticsearch</exclude>
            </excludes>
        </dependencySet>
    </dependencySets>
</assembly>
```
## 核心代码开发
### 解析器插件入口
根据说明文件，插件的入口类是`org.elasticsearch.plugin.analysis.rockstone.AnalysisRockstonePlugin`，其内容如下
```java

public class AnalysisRockstonePlugin extends Plugin implements AnalysisPlugin {

    @Override
    public Map<String, AnalysisModule.AnalysisProvider<AnalyzerProvider<? extends Analyzer>>> getAnalyzers() {
        return Collections.singletonMap("rockstone", RockstoneAnalyzerProvider::new);
    }
}
```
此次开发的解析器，需要继承`Plugin`类实现`AnalysisPlugin`接口，其它可实现的借口有`ActionPlugin`、`ClusterPlugin` 、`DiscoveryPlugin`、`IngestPlugin`、`MapperPlugin`、`NetworkPlugin`、`RepositoryPlugin`、`ScriptPlugin`、`SearchPlugin`、`ReloadablePlugin`。

在`AnalysisPlugin`接口中我们主要需要实现的方法有:
```java
public interface AnalysisPlugin {
    // 增加自定义CharFilters
    default Map<String, AnalysisProvider<CharFilterFactory>> getCharFilters() {
        return emptyMap();
    }

    // 增加自定义TokenFilters
    default Map<String, AnalysisProvider<TokenFilterFactory>> getTokenFilters() {
        return emptyMap();
    }

     // 增加自定义Tokenizers
    default Map<String, AnalysisProvider<TokenizerFactory>> getTokenizers() {
        return emptyMap();
    }

     // 增加自定义Analyzers
    default Map<String, AnalysisProvider<AnalyzerProvider<? extends Analyzer>>> getAnalyzers() {
        return emptyMap();
    }
}
```
在这里我们主要实现`getAnalyzers()`方法。该方法需要返回一个map，该map主要保存解析器名和提供解析器实现的映射。这里的注册方式参考了源生解析器的注册方式(`org.elasticsearch.indices.analysis.AnalysisModule`)。首先使用`RockstoneAnalyzerProvider`的构造方法实现接口`AnalysisModule.AnalysisProvider`的`T get()`方法，从而构造匿名类。`RockstoneAnalyzerProvider`类的实现主要参考`StandardAnalyzerProvider`如下所示
```java

public class RockstoneAnalyzerProvider extends AbstractIndexAnalyzerProvider<RockstoneAnalyzer> {

    private final RockstoneAnalyzer analyzer;

    public RockstoneAnalyzerProvider(IndexSettings indexSettings, Environment environment, String name, Settings settings) {
        // settings 中可以获取创建 analyzer时json中的配置，indexSetting可以拿到索引的配置
        super(indexSettings, name, settings);
        analyzer = new RockstoneAnalyzer();
    }

    @Override
    public RockstoneAnalyzer get() {
        return analyzer;
    }
}
```
该类可以继承抽象类`AbstractIndexAnalyzerProvider`，实现`get()` 方法即可。也可以自行实现`AnalyzerProvider<? extends Analyzer>`接口。该接口最重要是需要`T get()`方法，该方法需要返回`Analyzer`的子类，我们核心的业务功能就写在该类中。

### Analyzer
`Analyzer`主要作用是调用被final修饰的`tokenStream`方法返回`TokenStream`实例，它表了解析器如何从text解析生成terms。`Analyzer`的子类需要做的主要是实现`TokenStreamComponents createComponents(String)`方法来规定适用的Tokenizer和TokenFilter。该方法是必须是实现的。同时还能通过实现`Reader initReader(String fieldName, Reader reader)`来增加character filters。正好对应ES解析器分词的3个步骤。`Analyzer`提供TokenStream的源码如下。
```java
  
  public final TokenStream tokenStream(final String fieldName, final String text) {
    TokenStreamComponents components = reuseStrategy.getReusableComponents(this, fieldName);
    @SuppressWarnings("resource") final ReusableStringReader strReader = 
        (components == null || components.reusableStringReader == null) ?
        new ReusableStringReader() : components.reusableStringReader;
    strReader.setValue(text);
    final Reader r = initReader(fieldName, strReader); // 调用用户定值character filters 其本质是返回一个Reader
    if (components == null) {
      components = createComponents(fieldName); //获取用户定义的 Tokenizer和TokenFilter
      reuseStrategy.setReusableComponents(this, fieldName, components); 
    }

    components.setReader(r); // 默认是将reader通过Tokenizer::setReader赋值给分词器
    components.reusableStringReader = strReader;
    return components.getTokenStream();
  }
  
```

需要说明的是Tokenizer和TokenFilter都是TokenStream的子类，Tokenizer的输入时一个Reader，TokenFilter的输入是其它TokenStream。其嵌套关系决定了解析顺序。Tokenizer一般作为第一层。ES提供的`StandardAnalyzer`的`createComponents`方法是下如下
```java
  protected TokenStreamComponents createComponents(final String fieldName) {
    final StandardTokenizer src = new StandardTokenizer();
    src.setMaxTokenLength(maxTokenLength);
    TokenStream tok = new LowerCaseFilter(src);
    tok = new StopFilter(tok, stopwords);
    return new TokenStreamComponents(r -> {
      src.setMaxTokenLength(StandardAnalyzer.this.maxTokenLength);
      src.setReader(r);
    }, tok); // 构造TokenStreamComponents时传的lambda表达式会在 TokenStreamComponents实例调用setReader(r)时被运行
  }
```

如官方文档所说该解析器使用了`StandardTokenizer`以及`LowerCaseFilter`和`StopFilter`。

本次实现的解析器也是重写了`createComponents`和`initReader`方法。`createComponents`主要是使用了自定义的分词器，而在`initReader`中则使用了`StringReader`原因是默认的`Reader`没有实现`mark()`和`reset()`功能，代码如下。
```java
public class RockstoneAnalyzer extends Analyzer {
    @Override
    protected TokenStreamComponents createComponents(String fieldName) {
        return new TokenStreamComponents(new RockstoneTokenzier());
    }
    @Override
    protected Reader initReader(String fieldName, Reader reader) {
        StringBuilder text = new StringBuilder();
        try {
            int read = reader.read();
            while (read != -1) {
                text.append((char) read);
                read = reader.read();
            }
        } catch (IOException e) {
        }
        return new StringReader(text.toString());
    }
}
```

### TokenStream
在聊Tokenizer和TokenFilter之前先说下它们的父类TokenStream。TokenStream是一个抽象类，它用来将文档的某个属性或查询字符串转为一组Token。`TokenStream`继承自`AttributeSource`，父类为`TokenStream`提供访问Token属性的功能。一个TokenStream的工作流程基本如下：
1. 初始化`TokenStream`，从`AttributeSource`获取abttributes或向`AttributeSource`添加abttributes。
2. 使用者调用`reset()`方法
3. 使用者可以从`TokenStream`中取出Token的属性并且保存需要访问的Token属性
4. 使用者调用`incrementToken()`，直到返回false前都可以获得Token属性
5. 使用者调用`end()`执行结束`stream`的操作
6. 使用者在使用完`TokenStream`后调用`close()`来释放资源

通过对工作流的描述我们可以知道，在编写Tokenizer和TokenFilter时主要要复写的方法有`reset()`，`end()`，`close()`以及最重要的`incrementToken()`。`incrementToken()`方法返回bool值表示Token序列是否生成完。在调用一次该方法时，需要增量的生成1个Token(Tokenizer)或修改Token属性(TokenFilter)。

### Tokenizer
Tokenizer主要作用是使用传入的Reader生成Token，其子类在`bool incrementToken()`方法中需要生成Token属性。需要特别说明的是在生成新的Token时需要首先调用`AttributeSource#clearAttributes()`方法。下面我们看看ES对KeyWordTokenizer是如何实现的
```java
public final class KeywordTokenizer extends Tokenizer {
  // 默认的每个keyword最大的长度 
  public static final int DEFAULT_BUFFER_SIZE = 256; 

  private boolean done = false;
  private int finalOffset;
  // Token的term属性
  private final CharTermAttribute termAtt = addAttribute(CharTermAttribute.class);
  // Toke的偏移量
  private OffsetAttribute offsetAtt = addAttribute(OffsetAttribute.class);
  
  public KeywordTokenizer() {
    this(DEFAULT_BUFFER_SIZE);
  }

  public KeywordTokenizer(int bufferSize) {
    if (bufferSize > MAX_TOKEN_LENGTH_LIMIT || bufferSize <= 0) {
      throw new IllegalArgumentException("maxTokenLen must be greater than 0 and less than " + MAX_TOKEN_LENGTH_LIMIT + " passed: " + bufferSize);
    }
    termAtt.resizeBuffer(bufferSize);
  }

  public KeywordTokenizer(AttributeFactory factory, int bufferSize) {
    super(factory);
    if (bufferSize > MAX_TOKEN_LENGTH_LIMIT || bufferSize <= 0) {
      throw new IllegalArgumentException("maxTokenLen must be greater than 0 and less than " + MAX_TOKEN_LENGTH_LIMIT + " passed: " + bufferSize);
    }
    termAtt.resizeBuffer(bufferSize);
  }
  
  @Override
  public final boolean incrementToken() throws IOException {
    if (!done) {
      clearAttributes(); // 清空Token
      done = true;
      int upto = 0;
      char[] buffer = termAtt.buffer();
      while (true) {
        final int length = input.read(buffer, upto, buffer.length-upto); // 从input中读取char保存在该Token的term中
        if (length == -1) break;
        upto += length;
        if (upto == buffer.length)
          buffer = termAtt.resizeBuffer(1+buffer.length);
      }
      termAtt.setLength(upto); // 设置token长度
      finalOffset = correctOffset(upto);
      offsetAtt.setOffset(correctOffset(0), finalOffset); // Token设置偏移量
      return true;
    }
    return false;
  }
  
  @Override
  public final void end() throws IOException {
    super.end();
    // set final offset 
    offsetAtt.setOffset(finalOffset, finalOffset);
  }

  @Override
  public void reset() throws IOException {
    super.reset();
    this.done = false;
  }
}
```
由于keywordTokenizer就是把传入字符串原封不动作为1个term，所以在第一次while循环中会设置Token的CharTerm以及Offset属性然后返回false，接着第二次while循环，会因为input.read()返回-1跳出循环并返回false表示该字符串token处理结束。Token的属性除了term以及Offset属性，还可以设置很多其它属性，用的比较多的还有：`TypeAttribute`、 `PositionLengthAttribute`、 `PositionIncrementAttribute`。


本次需要实现的Tokenizer主要需要实现的功能是将"abcd"切分成\["abcd","bcd","cd","d"\]，思路是借助StringReader的`mark()`方法依次标记每个字符，每次将mark位置到最后的字符作为一个Token。本次自定义的Tokenizer与KeywordTokenizer基本相同，主要区别是`incrementToken()`方法，具体实现如下：
```java
 
 @Override
    public final boolean incrementToken() throws IOException {
        clearAttributes();
        int upto = 0;
        char[] buffer = termAtt.buffer();
        while (true) {
            int first = input.read();
            if (first == -1) break;
            // 为了标记,先读一个
            buffer[0] = (char) first;
            input.mark(1);
            upto++;
            // 取出其余部分
            final int length = input.read(buffer, upto, buffer.length - upto);
            // 再正常读 如果读到值则长度累加
            if (length != -1) {
                upto += length;
            }
            if (upto == buffer.length)
                buffer = termAtt.resizeBuffer(1 + buffer.length);
        }
        // 新增变量，在第一记录该字符串长度
        if (length == -1) {
            length = upto;
        }
        termAtt.setLength(upto);
        // 设置偏移量
        offsetAtt.setOffset(correctOffset(index), length);
        index++;
        // 将reader指针重置
        input.reset();
        if (length == (index - 1)) {
            return false;
        } else {
            return true;
        }
    }
```

### TokenFilter
TokenFilter比较好说，它将另一个TokenStream作为输入并在初始化时保存。在incrementToken()中一般先调用上一个TokenStream的incrementToken()，然后根据业务逻辑修改上一个TokenStream生成的Token属性。下面是`LowerCaseFilter`的源码
```java
public class LowerCaseFilter extends TokenFilter {
  private final CharTermAttribute termAtt = addAttribute(CharTermAttribute.class);
  
  public LowerCaseFilter(TokenStream in) {
      // 用另一个TokenStream实例化
      // 实例化时会将AttributeSource的主要变量于本实例绑定，这样也就可以直接用上一个TokenStream生成Token属性了
    super(in);
  }
  
  @Override
  public final boolean incrementToken() throws IOException {
    if (input.incrementToken()) {
        // 直接把charTerm属性转为小写
      CharacterUtils.toLowerCase(termAtt.buffer(), 0, termAtt.length());
      return true;
    } else
      return false;
  }
}

```

本次自定义插件未使用任何TokenFilter所以就不再列举其它Filter了。


### 测试
```java
   @Test
    public void testAnalyzer() throws Exception {
        RockstoneAnalyzer analyzer = new RockstoneAnalyzer();
        TokenStream ts = analyzer.tokenStream("text", "我爱北京 天安门"); // 获取自定义的TokenStream
        CharTermAttribute term = ts.addAttribute(CharTermAttribute.class); // 由于属性已近初始化所以直接获取CharTermAttribute的引用
        ts.reset();
        while (ts.incrementToken()) {
            System.out.println(term.toString());
        }
        ts.end();
        ts.close();
    }
    
--------- 输出 ---------
我爱北京 天安门
爱北京 天安门
北京 天安门
京 天安门
 天安门
天安门
安门
门
```
# 总结
此次本作者尝试以官方文档、源码及注释为主要学习材料，而不是学习他人总结的博客。优秀的开源组件都有详细的备注信息帮助开发理解代码，特别是可以扩展的代码以及核心代码。通过源码理解组件是个很好的学习方式，推荐大家尝试。
