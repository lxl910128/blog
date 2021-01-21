---
title: elasticsearch-ik插件学习（上）
date: 2021/01/21 16:26:10
categories:
- elasticsearch  

tags:
- elasticsearch
- es-plugin 

keywords:
- elasticsearch, plugin, ik, 插件, AnalysisIkPlugin, Tokenizer, 
---
# 概述
本系列主要从从源码的角度介绍es-IK插件，本文主要介绍ik是如何成为es分词插件的，核心是再复习一下es插件开发的一般流程。

<!-- more -->

# plugin-descriptor.properties

首先我们先从es插件的角度来看一下ik分词器的实现。根据之前[文章](https://blog.gaiaproject.club/es-develop-plugin/) 的介绍，我们知道es插件最终呈现给我们的都是一个zip包，这个包中有插件运行需要的jar包，资源文件以及`plugin-descriptor.properties`。这个配置文件是es官方要求必带的插件描述信息。ik插件的描述内容如下：

```properties
# 介绍
description=IK Analyzer for Elasticsearch
# 插件版本
version=6.5.0
# 插件名称
name=analysis-ik
# 主类，全路径
classname=org.elasticsearch.plugin.analysis.ik.AnalysisIkPlugin
# jvm版本
java.version=1.8
# elasticsearch版本
elasticsearch.version=6.5.0
```

从该文件可以看出，IK要求es加载的主类是`org.elasticsearch.plugin.analysis.ik.AnalysisIkPlugin`。

# AnalysisIkPlugin

源码如下：

```java
// 继承Plugin类，实现AnalysisPlugin接口，说明该插件时解析器插件
public class AnalysisIkPlugin extends Plugin implements AnalysisPlugin {

	public static String PLUGIN_NAME = "analysis-ik";
	// 提供2中分词方式，名称分别为ik_smart、ik_max_word
    @Override
    public Map<String, AnalysisModule.AnalysisProvider<TokenizerFactory>> getTokenizers() {
        Map<String, AnalysisModule.AnalysisProvider<TokenizerFactory>> extra = new HashMap<>();
				// value 是AnalysisProvider接口的匿名实现，
        // 该接口就一个方法需要实现，实现的方式是IkTokenizerFactory::getIkXXTokenizerFactory
        extra.put("ik_smart", IkTokenizerFactory::getIkSmartTokenizerFactory);
        extra.put("ik_max_word", IkTokenizerFactory::getIkTokenizerFactory);

        return extra;
    }
}
```

从源码中可以看出ik分词继承官方`Plugin`类实现`AnalysisPlugin`接口，确定了自己是解析器插件。从重写了`getTokenizers`方法可以看出ik插件主要提供了2个名为`ik_smart`和`ik_max_word`。

从返回的参数来看，es需要的是`AnalysisProvider`——解析器提供者接口的实现类。接口主要需要 实现的方法如下：

```java
  public interface AnalysisProvider<T> {

    /**
    * Creates a new analysis provider.
    * 返回1个新的解析器提供者
    * @param indexSettings 创建这个插件的索引的配置
    * @param environment   持久性存储中加载的节点env
    * @param name          插件名字
    * @param settings      没有上下文前缀的特定于组件的设置
    * @return a new provider instance  返回新的提供者实例
    */
 T get(IndexSettings indexSettings, Environment environment, String name, Settings settings) throws IOException;
}
```

`AnalysisProvider`提供了1个get方法用于创建一个`TokenizerFactory`，传入了各种配置参数，这些参数具体对应哪一块的配置，后续再补充。以`ik_max_word`分词器来看，这个get方法的实现对应的是`IkTokenizerFactory::getIkTokenizerFactory`下面我们看下这个方法，十分简单：

```java
public class IkTokenizerFactory extends AbstractTokenizerFactory {
  private Configuration configuration;
	// 构造函数
  public IkTokenizerFactory(IndexSettings indexSettings, Environment env, String name, Settings settings) {
      super(indexSettings, name, settings);
    // 插件主要用的参数是 index的配置 以及env
	  configuration=new Configuration(env,settings);
  }
	// ik_smart、ik_max_word的主要区别就是 配置中是否将setUseSmart设为ture
  public static IkTokenizerFactory getIkTokenizerFactory(IndexSettings indexSettings, Environment env, String name, Settings settings) {
      return new IkTokenizerFactory(indexSettings,env, name, settings).setSmart(false);
  }

  public static IkTokenizerFactory getIkSmartTokenizerFactory(IndexSettings indexSettings, Environment env, String name, Settings settings) {
      return new IkTokenizerFactory(indexSettings,env, name, settings).setSmart(true);
  }

  public IkTokenizerFactory setSmart(boolean smart){
        this.configuration.setUseSmart(smart);
        return this;
  }
	
  @Override
  public Tokenizer create() {
    // 常见IK分词器
      return new IKTokenizer(configuration);  }
}
```

可以看出来官方其实已经提供了`AbstractTokenizerFactory`，抽象的分词工厂，这个分词工厂实现了`TokenizerFactory`接口，这个接口就1个方法` Tokenizer create()`创建分词器，这里我们可以看见，我们用`Configuration`创建了IK分词器。而`AbstractTokenizerFactory`主要做的通用工作是提供了以下几个方法：`version()`获取版本，` Index index()`返回index对象，`IndexSettings getIndexSettings()`返回index配置信息，这些方法可以在`TokenizerFactory`直接使用，但是IK分词器并没有用到，主要就用`IndexSettings`和`Environment`构造了自己用的配置对象`Configuration`

# IKTokenizer

下面我们看看`IKTokenizer`的代码，这就是分词器逻辑的实现，代码如下：

```java
public final class IKTokenizer extends Tokenizer {
	
	//IK分词器实现
	private IKSegmenter _IKImplement;
	
	//词元文本属性
	private final CharTermAttribute termAtt;
	//词元位移属性
	private final OffsetAttribute offsetAtt;
	//词元分类属性（该属性分类参考org.wltea.analyzer.core.Lexeme中的分类常量）
	private final TypeAttribute typeAtt;
	//记录最后一个词元的结束位置
	private int endPosition;
   	private int skippedPositions;
   	private PositionIncrementAttribute posIncrAtt;
    /**
	 * Lucene 4.0 Tokenizer适配器类构造函数
     */
	public IKTokenizer(Configuration configuration){
	    super();
	    offsetAtt = addAttribute(OffsetAttribute.class);
	    termAtt = addAttribute(CharTermAttribute.class);
	    typeAtt = addAttribute(TypeAttribute.class);
        posIncrAtt = addAttribute(PositionIncrementAttribute.class);
		// 用Tokenizer中的input初始化 ik实际分词器
        _IKImplement = new IKSegmenter(input,configuration);
	}

	/* (non-Javadoc)
	 * @see org.apache.lucene.analysis.TokenStream#incrementToken()
	 */
	@Override
	public boolean incrementToken() throws IOException {
		//清除所有的词元属性
		clearAttributes();
        skippedPositions = 0;

        Lexeme nextLexeme = _IKImplement.next();
		if(nextLexeme != null){
            posIncrAtt.setPositionIncrement(skippedPositions +1 );

			//将Lexeme转成Attributes
			//设置词元文本
			termAtt.append(nextLexeme.getLexemeText());
			//设置词元长度
			termAtt.setLength(nextLexeme.getLength());
			//设置词元位移
  offsetAtt.setOffset(correctOffset(nextLexeme.getBeginPosition()), correctOffset(nextLexeme.getEndPosition()));

      //记录分词的最后位置
			endPosition = nextLexeme.getEndPosition();
			//记录词元分类
			typeAtt.setType(nextLexeme.getLexemeTypeString());			
			//返会true告知还有下个词元
			return true;
		}
		//返会false告知词元输出完毕
		return false;
	}
	
	/*
	 * 在开始消费 incrementToken 前调用该方法
   * 作用是：将此流重置为干净状态。 有状态的分词器的实现必须实现此方法，以便可以重用分词器，就像它们是重新创建一样。
   * 特别注意 rest时设置了新的input，这表明准备开始分新的1个词！
	 */
	@Override
	public void reset() throws IOException {
		super.reset();
		_IKImplement.reset(input);
        skippedPositions = 0;
	}	
	// incrementToken 返回false时调用end，也就是1个熟悉分词结束后调用end
	@Override
	public final void end() throws IOException {
        super.end();
	    // set final offset
		int finalOffset = correctOffset(this.endPosition);
		offsetAtt.setOffset(finalOffset, finalOffset);
        posIncrAtt.setPositionIncrement(posIncrAtt.getPositionIncrement() + skippedPositions);
	}
}
```

这里重点注意`incrementToken()`方法，在具体分词1个属性时，会重复调用该方法，直到方法返回false表示这个属性分词完毕，至于分词的结果有：词元文本（charTermAttribute）、词元位置（OffsetAttribute）、词元分类（TypeAttribute）等直接设置即可，因为使用了` addAttribute(XXXXAttribute.class);`所有es会自动消费这些值。注意这里从感觉是就是用新分词属性覆盖老的数据，es设计就是这样。分出来的元素放到这些属性里，你不能管消费者（IndexWriter）如何使用。

官方对incrementToken方法的注释放在这，如果要写分词插件，深入理解这个方法是如何被es使用的十分重要。

```
Consumers (i.e., IndexWriter) use this method to advance the stream to the next token. 
使用者（如 IndexWriter）使用这个方法从stream中生成下一个token
Implementing classes must implement this method and update the appropriate AttributeImpls with the attributes of the next token.
实现类必须实现这个方法，使用下一个token的属性更新对应的AttributeImpls（调用addAttribute()方法注册的属性）
The producer must make no assumptions about the attributes after the method has been returned: the caller may arbitrarily change it.
incrementToken返回后，分词器不能对属性有任何假设，调用者可以任意改变它，即方法返回分词器就不要在读取属性值了，很可能已经变了
If the producer needs to preserve the state for subsequent calls, it can use captureState to create a copy of the current attribute state.
如果生产者需要保留该状态以用于后续调用，则可以使用captureState创建当前属性状态的副本。
This method is called for every token of a document, so an efficient implementation is crucial for good performance. 
对于文档的每个token都调用此方法，因此有效的实现对于良好的性能至关重要。
To avoid calls to addAttribute(Class) and getAttribute(Class), references to all AttributeImpls that this stream uses should be retrieved during instantiation.
为了避免调用addAttribute（Class）和getAttribute（Class），初始化期间应该设置好所有这个stream使用到的AttributeImpls，也就是说，在初始化分词插件时，需要调用addAttribute，把可能分出的属性都设置好。
To ensure that filters and consumers know which attributes are available, the attributes must be added during instantiation. 
为了确保过滤器和使用者知道哪些属性可用，必须在实例化期间添加属性（调用addAttribute）。
Filters and consumers are not required to check for availability of attributes in incrementToken().
不需要过滤器和使用者检查incrementToken（）中属性的可用性。
```

整体看下来，分1个属性，主要是连续调用`incrementToken()`方法，在调用前会先调用`rest()`设置输入流，调用后（返回false 表示结束）则会调用`end()`方法。在ik分词器中生成token属性的主要是`IKSegmenter`，他是ik分词算法实现的核心逻辑，也是我们下一部分关注的重点。




