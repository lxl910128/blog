---
title: elasticsearch学习——Lucene  
date: 2020/10/06 13:10:10
categories:
- elasticsearch  

tags:
- elasticsearch
- Lucene  

keywords:
- elasticsearch, lucene, inverted index, segment, segment, Finite State Transducers
---

# 概要
elasticsearch是基于Lucene实现的分布式查询分析引擎。它的主要工作是在lucene上做了分布式的封装，增强了可用性及可扩展性（封装的相当成功）。要想学好elasticsearch，那么lucene都是绕不过去的。
Lucene是基于java实现的全文搜索引擎。它不是完整的应用，但是提供了开源代码及API。  

本文将从以下几个方面对Lucene进行简单介绍：首先通过代码介绍了lucene如何索引文档和搜索文档，然后介绍了Lucene设计的基本概念，接着粗略介绍了文档和搜索的过程，最后从底层存储结构出发介绍了Lucene存储了哪些索引。  

本文介绍的lucene的版本为7.X，是elasticsearch 6.X使用的版本。
<!--more-->
# 代码示例
Lucene最核心的2个功能就是，索引文档以及搜索被索引的文档。下面给出这两个功能demo。

## 索引文档
```java
@Test
    public void createIndexTest() throws Exception {
        //1. 采集数据
        // Sku 为商品数据
        SkuDao skuDao = new SkuDaoImpl();
        List<Sku> skuList = skuDao.querySkuList();

        //文档集合
        List<Document> docList = new ArrayList<>();

        for (Sku sku : skuList) {
            //2. 创建文档对象
            Document document = new Document();

            //创建域对象并且放入文档对象中
            /**
             * 是否分词: 否, 因为主键分词后无意义
             * 是否索引: 是, 如果根据id主键查询, 就必须索引
             * 是否存储: 是, 因为主键id比较特殊, 可以确定唯一的一条数据, 在业务上一般有重要所用, 所以存储
             *      存储后, 才可以获取到id具体的内容
             */
            document.add(new StringField("id", sku.getId(), Field.Store.YES));

            /**
             * 是否分词: 是, 因为名称字段需要查询, 并且分词后有意义所以需要分词
             * 是否索引: 是, 因为需要根据名称字段查询
             * 是否存储: 是, 因为页面需要展示商品名称, 所以需要存储
             */
            document.add(new TextField("name", sku.getName(), Field.Store.YES));

            /**
             * 是否分词: 是(因为lucene底层算法规定, 如果根据价格范围查询, 必须分词)
             * 是否索引: 是, 需要根据价格进行范围查询, 所以必须索引
             * 是否存储: 是, 因为页面需要展示价格
             */
            document.add(new IntPoint("price", sku.getPrice()));
            document.add(new StoredField("price", sku.getPrice()));

            /**
             * 是否分词: 否, 因为不查询, 所以不索引, 因为不索引所以不分词
             * 是否索引: 否, 因为不需要根据图片地址路径查询
             * 是否存储: 是, 因为页面需要展示商品图片
             */
            document.add(new StoredField("image", sku.getImage()));

            /**
             * 是否分词: 否, 因为分类是专有名词, 是一个整体, 所以不分词
             * 是否索引: 是, 因为需要根据分类查询
             * 是否存储: 是, 因为页面需要展示分类
             */
            document.add(new StringField("categoryName", sku.getCategoryName(), Field.Store.YES));

            /**
             * 品牌名称
             * 是否分词: 否, 因为品牌是专有名词, 是一个整体, 所以不分词
             * 是否索引: 是, 因为需要根据品牌进行查询
             * 是否存储: 是, 因为页面需要展示品牌
             */
            document.add(new StringField("brandName", sku.getBrandName(), Field.Store.YES));

            //将文档对象放入到文档集合中
            docList.add(document);
        }
        //3. 创建分词器, StandardAnalyzer标准分词器, 对英文分词效果好, 对中文是单字分词, 也就是一个字就认为是一个词.
        Analyzer analyzer = new StandardAnalyzer();
        //4. 创建Directory目录对象, 目录对象表示索引库的位置
        Directory  dir = FSDirectory.open(Paths.get("E:\\dir"));
        //5. 创建IndexWriterConfig对象, 这个对象中指定切分词使用的分词器
        IndexWriterConfig config = new IndexWriterConfig(analyzer);
        //6. 创建IndexWriter输出流对象, 指定输出的位置和使用的config初始化对象
        IndexWriter indexWriter = new IndexWriter(dir, config);
        //7. 写入文档到索引库
        for (Document doc : docList) {
            indexWriter.addDocument(doc);
        }
        //8. 释放资源
        indexWriter.close();
    }
```

# 搜索文档
```java
 @Test
    public void testIndexSearch() throws Exception {

        //1. 创建分词器(对搜索的关键词进行分词使用)
        //注意: 分词器要和创建索引的时候使用的分词器一模一样
        Analyzer analyzer = new StandardAnalyzer();

        //2. 创建查询对象,
        //第一个参数: 默认查询域, 如果查询的关键字中带搜索的域名, 则从指定域中查询, 如果不带域名则从, 默认搜索域中查询
        //第二个参数: 使用的分词器
        QueryParser queryParser = new QueryParser("name", analyzer);

        //3. 设置搜索关键词
        //华 OR  为   手   机
        Query query = queryParser.parse("华为手机");

        //4. 创建Directory目录对象, 指定索引库的位置
        Directory dir = FSDirectory.open(Paths.get("E:\\dir"));
        //5. 创建输入流对象
        IndexReader indexReader = DirectoryReader.open(dir);
        //6. 创建搜索对象
        IndexSearcher indexSearcher = new IndexSearcher(indexReader);
        //7. 搜索, 并返回结果
        //第二个参数: 是返回多少条数据用于展示, 分页使用
        TopDocs topDocs = indexSearcher.search(query, 10);

        //获取查询到的结果集的总数, 打印
        System.out.println("=======count=======" + topDocs.totalHits);

        //8. 获取结果集
        ScoreDoc[] scoreDocs = topDocs.scoreDocs;

        //9. 遍历结果集
        if (scoreDocs != null) {
            for (ScoreDoc scoreDoc : scoreDocs) {
                //获取查询到的文档唯一标识, 文档id, 这个id是lucene在创建文档的时候自动分配的
                int  docID = scoreDoc.doc;
                //通过文档id, 读取文档
                Document doc = indexSearcher.doc(docID);
                System.out.println("==================================================");
                //通过域名, 从文档中获取域值
                System.out.println("===id==" + doc.get("id"));
                System.out.println("===name==" + doc.get("name"));
                System.out.println("===price==" + doc.get("price"));
                System.out.println("===image==" + doc.get("image"));
                System.out.println("===brandName==" + doc.get("brandName"));
                System.out.println("===categoryName==" + doc.get("categoryName"));

            }
        }
        //10. 关闭流
    }

```

# 基本概念
通过上述示例，相信大家对Lucene有了一个大概的理解，下面我们来介绍Lucene的一些基础概念。
* 正向索引（forward index）：每个文件都对应一个文件ID，文件内容被表示为一系列关键词的集合。建立文档ID到该文档关键字的映射就是正向索引。
* 倒排索引（inverted index）：建立关键词到文档的映射。倒排索引一般由字典表和倒排索列表构成。  
    * 字典表：单词词典是由文档集合中出现过的所有单词构成的字符串集合，单词词典内每条索引项记载单词本身的一些信息以及指向“倒排列表”的指针。
    * 倒排表：记载了出现过某个单词的所有文档的列表及单词在该文档中出现的位置信息，每条记录称为一个倒排项(Posting)。根据倒排列表，即可获知哪些文档包含某个单词。
* 索引（index）：一个索引包含一系列文档，对应elasticsearch索引的概念，类似于传统关系数据库的表。
* 文档（document）：一个文档包含一系列属性，对应elasticsearch文档的概念，类似于传统关系数据库的一行数据。
* 属性（field）：属性是被命名的一系列terms，对应elasticsearch属性的概念，类似于传统关系数据库列的概念。
* term：索引的最小单位，是经过词法分词和语言处理后的字符串，对应elasticsearch term的概念，类似于传统关系数据库某一个column的值。
* 属性类型（type of field）：用于定义某field存入lucene的方式，主要从是否分词、是否能搜索、是否存存储三个维度定义。
* 段（segment）：Lucene索引落在磁盘中的数据结构称为段，由一些列数据文件组成。每个段是一个完整且独立的索引，可支持相关查询功能，每个索引由多个段组成。在索引数据时，数据先保存在内存中，等积累一定数量后写入磁盘形成段。索引定期会将小的段合并成大的段。

# 核心流程
正如前面的文本文件搜索程序所示，Lucene的信息检索功能主要包含两个主要流程：索引和搜索。这两部分的整体流程如下：

![索引和搜搜过程](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-1.png)

* 索引过程
    * 对待索引的文档进行分词处理（1）
    * 结合分词处理的结果，建立索引（字典表、倒排表等）（2）
    * 将倒排索引写入索引存储（3）、（4）
* 查询过程
    * 对用户的查询语句进行分词处理（a）
    * 搜索索引得到结果文档集，其中涉及到从索引存储中加载索引到内存的过程（b）、（c）
    * 对搜索结果进行排序并返回结果（d）、（e）
 
## 索引流程
1. 将待索引的文档传递给分词器进行处理，我们样例程序中的StandardAnalyzer即为标准英文分词器，如果需要中文分词，可以使用开源界贡献的插件ik或自定义。
2. 分词过程会把文档拆分成一个个独立的词（Term），期间会去除标点符号和停用词（“the”、“this”、“a”...），并对词做小写化等处理。
3. 建立词典表和倒排表，如下图
![倒排表和索引表](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-2.png)

4. Lucene为了加快索引速度，采用了LSM Tree结构，先把索引数据缓存在内存。
![LSM-TREE](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-3.png)

5. 当内存空间占用较高或达到时间限制后，内存中的数据会被写入磁盘形成一个数据段（segment），segment实际包含词典、倒排表、字段数据等等多个文件。
![字典倒排表2](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-4.png)

## 搜索流程
1. 对用户的请求语句进行词法、语法分析，生成查询语法树，把文本请求转换为Lucene理解的请求对象。
2. 按照查询语法树，搜索索引获取最终匹配的文档id集合
3. 对查询结果进行打分排序，获取Top N的文档id集合，获取文档原始数据后返回用户。影响打分的因数因素包含：
    * 词频/文档频率（TF/IDF）：词频越高打分越高，文档频率越高打分越低
    * boost：lucene支持针对不同字段设置权重，例如当Term出现在标题字段时的打分，通常高于其出现在文档内容中的打分。
    * 等

# 核心存储结构
segment中存储的索引文件有以下这些。  
![段文件](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-5.png)  

下面对各分类做简要说明。

## 倒排索引——字典表
字典表在倒排索引中至关重要。它根据给定的term找到该term所对应的倒排文档id。可实现此功能的算法主要有如下这些：
* 排序列表Array/List：使用二分法查找，不平衡
* HashMap/TreeMap：性能高，内存消耗大，几乎是原始数据的三倍
* skip table：跳跃表，可快速查找词语，在lucene、redis、Hbase等均有实现。相对于TreeMap等结构，特别适合高并发场景，但无法模糊查询。Lucene 4.0前字典表主要使用该技术，后换成了FST，但跳跃表在Lucene其他地方还有应用如倒排表合并和文档号索引。
* Trie：适合英文词典，如果系统中存在大量字符串且这些字符串基本没有公共前缀，则相应的trie树将非常消耗内存
* Double Array Trie：适合做中文词典，内存占用小，很多分词工具均采用此种算法
* Ternary Search Tree：三叉树，每一个node有3个节点，兼具省空间和查询快的优点
* Finite State Transducers (FST)：一种有限状态转移机，Lucene 4有开源实现，并大量使用。FST的理论基础：《Direct constructionofminimal acyclic subsequential transducers》，通过输入有序字符串构建最小有向无环图。**复杂度O(len(str))**。**优点**：内存占用率低，压缩率一般在3倍~20倍之间、模糊查询支持好、查询快。**缺点**：结构复杂、输入要求有序、更新不易。  

在Lucene中`.tip`和`.tim`形成FST索引。在tip中每列包含一个FST索引，所以会有多个FST，每个FST存放前缀和后缀块指针。tim里面存放后缀块和词的其他信息如倒排表指针、TFDF等，`.doc、.pos、.pay`共同组成每个单词的倒排表。  
![FST倒排索引.png](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-6.png)

Lucene中查询倒排索引的一般步骤为：
1. 内存加载tip文件，通过FST匹配前缀找到后缀词块位置。
2. 根据词块位置，读取磁盘中tim文件中后缀块并找到后缀和相应的倒排表位置信息。
3. 根据倒排表位置去doc文件中加载倒排表。

倒排表就是文档号集合，但怎么存，怎么取也有很多讲究，Lucene现使用的倒排表结构叫[Frame of reference](https://www.elastic.co/blog/frame-of-reference-and-roaring-bitmaps),它主要有两个特点:**数据压缩**，可以看下图怎么将6个数字从原先的24bytes压缩到7bytes。**跳跃表加速合并**，因为布尔查询时，and 和or 操作都需要合并倒排表，这时就需要快速定位相同文档号，所以利用跳跃表来进行相同文档号查找。  
![倒排表压缩](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-7.png)

## 正向索引——行式存储
正向文件指的就是原始文档，Lucene对原始文档也提供了存储功能，它存储特点就是分块+压缩，`.fdt`文件就是存放原始文档的文件，它占了索引库90%的磁盘空间，`.fdx`文件为索引文件，通过文档号（自增数字）快速得到文档位置，它们的文件结构如下

![正向行式索引](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-8.png)  

`.fnm`中为元信息存放了各列类型、列名、存储方式等信息。  
![fnm](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-9.png) 

`.fdt`为文档值，里面一个chunk就是一个块，Lucene索引文档时，先缓存文档，缓存大于16KB时，就会把文档压缩存储。一个chunk包含了该chunk起始文档、多少个文档、压缩后的文档内容。  
![fdt](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-10.png)

`.fdx`为文档号索引，倒排表存放的时文档号，通过fdx才能快速定位到文档位置即chunk位置，它的索引结构比较简单，就是跳跃表结构，首先它会把1024个chunk归为一个block,每个block记载了起始文档值，block就相当于一级跳表。  
![fdx](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-11.png) 

所以查找文档，就分为三步：
1. 二分查找block，定位属于哪个block。
2. 根据从block里根据每个chunk的起始文档号，找到属于哪个chunk和chunk位置。
3. 去加载fdt的chunk，找到文档。这里还有一个细节就是存放chunk起始文档值和chunk位置不是简单的数组，而是采用了平均值压缩法。所以第N个chunk的起始文档值由 DocBase + AvgChunkDocs * n + DocBaseDeltas[n]恢复而来，而第N个chunk再fdt中的位置由 StartPointerBase + AvgChunkSize * n + StartPointerDeltas[n]恢复而来。

从上面分析可以看出，lucene对原始文件的存放是行是存储，并且为了提高空间利用率，是多文档一起压缩，因此取文档时需要读入和解压额外文档，因此取文档过程非常依赖随机IO，以及lucene虽然提供了取特定列，但从存储结构可以看出，并不会减少取文档时间。

## 正向索引——列式存储
我们知道倒排索引能够解决从词到文档的快速映射，但当我们需要对检索结果进行分类、排序、数学计算等聚合操作时需要文档号到值的快速映射，而原先不管是倒排索引还是行式存储的文档都无法满足要求。  

原先4.0版本之前，Lucene实现这种需求是通过FieldCache，它的原理是通过按列逆转倒排表将（field value ->doc）映射变成（doc -> field value）映射,但这种实现方法有着两大显著问题：构建时间长；内存占用大，易OutOfMemory，且影响垃圾回收。  

因此4.0版本后Lucene推出了DocValues来解决这一问题，它和FieldCache一样，都为列式存储，但它有如下优点：预先构建，写入文件；基于映射文件来做，脱离JVM堆内存，系统调度缺页。  

Lucene目前有五种类型的DocValues：NUMERIC、BINARY、SORTED、SORTED_SET、SORTED_NUMERIC，针对每种类型Lucene都有特定的压缩方法。如对NUMERIC类型即数字类型，数字类型压缩方法很多，如：增量、表压缩、最大公约数，根据数据特征选取不同压缩方法。SORTED类型即字符串类型，压缩方法就是表压缩：预先对字符串字典排序分配数字ID，存储时只需存储字符串映射表，和数字数组即可，而这数字数组又可以采用NUMERIC压缩方法再压缩。这样就将原先的字符串数组变成数字数组，一是减少了空间，文件映射更有效率，二是原先变成访问方式变成固长访问。  

对DocValues的应用，ElasticSearch功能实现地更系统、更完整，即ElasticSearch的Aggregations——聚合功能，它的聚合功能分为三类：
1. Metric（统计），典型功能：sum、min、max、avg、cardinality、percent等
2. Bucket（分桶），典型功能：日期直方图，分组，地理位置分区
3. Pipline（基于聚合再聚合），典型功能：基于各分组的平均值求最大值。  

基于这些聚合功能，ElasticSearch不再局限与检索，而能够回答诸如"销售部门男女人数、平均年龄是多少"的问题。实现过程如下图  
![DocValues](https://rfc2616.oss-cn-beijing.aliyuncs.com/blog/Lucene-12.png)

基本思路如下：
1. 从倒排索引中找出销售部门的倒排表。
2. 根据倒排表去性别的DocValues里取出每个人对应的性别，并分组到Female和Male里。
3. 根据分组情况和年龄DocValues，计算各分组人数和平均年龄
4. 因为ElasticSearch是分区的，所以对每个分区的返回结果进行合并就是最终的结果。

上面就是ElasticSearch进行聚合的整体流程，也可以看出ElasticSearch做聚合的一个瓶颈就是最后一步的聚合只能单机聚合，也因此一些统计会有误差，比如count(*) group by producet limit 5,最终总数不是精确的。因为单点内存聚合，所以每个分区不可能返回所有分组统计信息，只能返回部分，汇总时就会导致最终结果不正确。

# 参考
https://www.bilibili.com/video/BV1eJ411q7nw  
https://cloud.tencent.com/developer/article/1181441  
https://lucene.apache.org/core/7_7_3/core/org/apache/lucene/codecs/lucene70/package-summary.html#package.description  
https://www.jianshu.com/p/2748af48e036  
https://blog.csdn.net/njpjsoftdev/article/details/54015485
