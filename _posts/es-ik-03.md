---
title: elasticsearch-ik插件学习（下）
date: 2021/01/30 16:26:10
categories:
- elasticsearch  

tags:
- elasticsearch
- es-plugin 

keywords:
- elasticsearch, plugin, ik, 插件, IKSegmenter, AnalyzeContext, LetterSegmenter, IKArbitrator
---
# 概述
本系列主要从源码的角度介绍es-IK插件，本文主要从源码层面介绍ik分词中中文自分词器的实现逻辑。

<!-- more -->

# CJKSegmenter

首先我们回顾下IK分词器的主要逻辑是使用不同的子分词器对同一句话进行分词，然后（配置smart模式）使用一个歧义判断器得到最种的分词结果。今天我们就看看我们最常用的中日韩子分词器`CJKSegmenter`，与之前介绍的`LetterSegmenter`一样，它主要是实现了`ISegmenter`接口。该接口的2个重要方式法是`void analyze(AnalyzeContext context)`，对`context`中`segmentBuff`进行分词，产生N个词元`Lexeme`放在原始词元`orgLexemes`数组中，等待歧义裁决器的确定。以及`void reset()`方法，重置分词器状态，为分新的词做准备。那么`CJKSegmenter`是怎么实现的呢？上源码：

```java
class CJKSegmenter implements ISegmenter {
    //子分词器标签
    static final String SEGMENTER_NAME = "CJK_SEGMENTER";
    //待处理的分词hit队列
    private List<Hit> tmpHits;
    CJKSegmenter() {
        this.tmpHits = new LinkedList<Hit>();
    }
    @Override
	public void analyze(AnalyzeContext context) {
        if (CharacterUtil.CHAR_USELESS != context.getCurrentCharType()) {
            //优先处理tmpHits中的hit
            if (!this.tmpHits.isEmpty()) {
                //处理词段队列
                Hit[] tmpArray = this.tmpHits.toArray(new Hit[this.tmpHits.size()]);
                for (Hit hit : tmpArray) {
                    hit = Dictionary.getSingleton().matchWithHit(context.getSegmentBuff(), context.getCursor(), hit);
                    if (hit.isMatch()) {
                        //输出当前的词
                        Lexeme newLexeme = new Lexeme(context.getBufferOffset(), hit.getBegin(), context.getCursor() - hit.getBegin() + 1, Lexeme.TYPE_CNWORD);
                        context.addLexeme(newLexeme);
                        
                        if (!hit.isPrefix()) {//不是词前缀，hit不需要继续匹配，移除
                            this.tmpHits.remove(hit);
                        }
                        
                    } else if (hit.isUnmatch()) {
                        //hit不是词，移除
                        this.tmpHits.remove(hit);
                    }
                }
            }
            //*********************************
            //再对当前指针位置的字符进行单字匹配
            Hit singleCharHit = Dictionary.getSingleton().matchInMainDict(context.getSegmentBuff(), context.getCursor(), 1);
            if (singleCharHit.isMatch()) {//首字成词
                //输出当前的词
                Lexeme newLexeme = new Lexeme(context.getBufferOffset(), context.getCursor(), 1, Lexeme.TYPE_CNWORD);
                context.addLexeme(newLexeme);
                //同时也是词前缀
                if (singleCharHit.isPrefix()) {
                    //前缀匹配则放入hit列表
                    this.tmpHits.add(singleCharHit);
                }
            } else if (singleCharHit.isPrefix()) {//首字为词前缀
                //前缀匹配则放入hit列表
                this.tmpHits.add(singleCharHit);
            }
        } else {
            //遇到CHAR_USELESS字符
            //清空队列
            this.tmpHits.clear();
        }
        //判断缓冲区是否已经读完
        if (context.isBufferConsumed()) {
            //清空队列
            this.tmpHits.clear();
        }
        //判断是否锁定缓冲区
        if (this.tmpHits.size() == 0) {
            context.unlockBuffer(SEGMENTER_NAME);
        } else {
            context.lockBuffer(SEGMENTER_NAME);
        }
    }
    public void reset() {
        //清空队列
        this.tmpHits.clear();
    }
}
```

中文分词器的主逻辑十分简单，每次触发分词时首先判断`tmpHits`，待处理hit中是否有值，如果有值则先处理，处理的方式是调用字典类`Dictionary`的匹配函数`matchWithHit`，如果匹配则用hit产生一个`Lexeme`，如果这个词还是其他的词的前缀，则继续保留在`tmpHits`中，否则移除。剩余的hit处理完后再对当前指针位置的字符进行单字匹配。同样是借助字典类`Dictionary`，判断首字符是否成词，如果成词，则将hit转为Lexeme并保存，如果这个词还是前缀词，则存在tmpHits中等待下次匹配看是否能与后文成为更长的词。这是一次分词就结束了。当然最后还做了些收尾工作，比如如果下个字符是CHAR_USELESS（停止词？）则清空tmpHits，如果分词结束也需要清tmpHits，如果tmpHits有词，即这个分词器是有状态的则需要上锁，防止多线程的情况下出现问题。下面我们重点看看`Dictionary`的结构。

# Dictionary

ik中文分词器核心的字典分词对象，首先我们看看他的主要属性和构造方法。

```java
/**
 * 词典管理类,单例模式
 */
public class Dictionary {
  /*
	 * 词典单子实例
	 */
	private static Dictionary singleton;
  // 3个子字典
	private DictSegment _MainDict;// 主字典
	private DictSegment _QuantifierDict;// 量词字典
	private DictSegment _StopWords;// 停止词字典
  private Configuration configuration;

	private static final Logger logger = ESPluginLoggerFactory.getLogger(Monitor.class.getName());
  // 监控动态字典的线程
	private static ScheduledExecutorService pool = Executors.newScheduledThreadPool(1);
	// 各种字典的文件名
	private static final String PATH_DIC_MAIN = "main.dic";
	private static final String PATH_DIC_SURNAME = "surname.dic";
	private static final String PATH_DIC_QUANTIFIER = "quantifier.dic";
	private static final String PATH_DIC_SUFFIX = "suffix.dic";
	private static final String PATH_DIC_PREP = "preposition.dic";
	private static final String PATH_DIC_STOP = "stopword.dic";
  // 各种配置
	private final static  String FILE_NAME = "IKAnalyzer.cfg.xml";
	private final static  String EXT_DICT = "ext_dict";
	private final static  String REMOTE_EXT_DICT = "remote_ext_dict";
	private final static  String EXT_STOP = "ext_stopwords";
	private final static  String REMOTE_EXT_STOP = "remote_ext_stopwords";
  //  ${es-config}/analysis-ik/
	private Path conf_dir;
	// 插件配置文件IKAnalyzer.cfg.xml的内容
	// 主要用于配置用户自己加的字典，这种字典主要是普通字典和停用词字典，可以用本地和远程2中方式
	private Properties props;
  // 私有构造方法 主要找配置文件IKAnalyzer.cfg.xml的路径，加载配置文件
  private Dictionary(Configuration cfg) {
		this.configuration = cfg;
		this.props = new Properties();
		// ${es-config}/analysis-ik/ 或 在插件文件加中找
		this.conf_dir = cfg.getEnvironment().configFile().resolve(AnalysisIkPlugin.PLUGIN_NAME);
		// configFile = ${es-config}/analysis-ik/IKAnalyzer.cfg.xml
		Path configFile = conf_dir.resolve(FILE_NAME);
		InputStream input = null;
		try {
			logger.info("try load config from {}", configFile);
			input = new FileInputStream(configFile.toFile());
		} catch (FileNotFoundException e) {
			// 在插件文件加中找
			conf_dir = cfg.getConfigInPluginDir();
			configFile = conf_dir.resolve(FILE_NAME);
			try {
				logger.info("try load config from {}", configFile);
				input = new FileInputStream(configFile.toFile());
			} catch (FileNotFoundException ex) {
				// We should report origin exception
				logger.error("ik-analyzer", e);
			}
		}
		if (input != null) {
			try {
				// load属性
				props.loadFromXML(input);
			} catch (IOException e) {
				logger.error("ik-analyzer", e);
			}
		}
	}
  /**
	 * 词典初始化 由于IK Analyzer的词典采用Dictionary类的静态方法进行词典初始化
	 * 只有当Dictionary类被实际调用时，才会开始载入词典， 这将延长首次分词操作的时间 该方法提供了一个在应用加载阶段就初始化字典的手段
	 */
	public static synchronized void initial(Configuration cfg) {
		if (singleton == null) {
			synchronized (Dictionary.class) {
				if (singleton == null) {
          // 构建Dictionary 
					singleton = new Dictionary(cfg);
					singleton.loadMainDict();// 本地主字典
					singleton.loadSurnameDict();// 本地姓字典 加载了个寂寞并没用
					singleton.loadQuantifierDict();// 本地量词字典
					singleton.loadSuffixDict(); // 本地后缀字典 加载了个寂寞并没用
					singleton.loadPrepDict(); // 本地介词字典 加载了个寂寞并没用
					singleton.loadStopWordDict(); // 本地停用词字典
					if(cfg.isEnableRemoteDict()){
						// 建立监控线程
						for (String location : singleton.getRemoteExtDictionarys()) {
							// 10 秒是初始延迟可以修改的 60是间隔时间 单位秒
							pool.scheduleAtFixedRate(new Monitor(location), 10, 60, TimeUnit.SECONDS);
						}
						for (String location : singleton.getRemoteExtStopWordDictionarys()) {
							pool.scheduleAtFixedRate(new Monitor(location), 10, 60, TimeUnit.SECONDS);
						}
					}
				}
			}
		}
	}
  // 加载主词典及扩展词典
	private void loadMainDict() {
		// 建立一个主词典实例
		_MainDict = new DictSegment((char) 0);
		// 读取主词典文件
		Path file = PathUtils.get(getDictRoot(), Dictionary.PATH_DIC_MAIN);
		loadDictFile(_MainDict, file, false, "Main Dict");
		// 加载扩展词典
		this.loadExtDict();
		// 加载远程自定义词库
		this.loadRemoteExtDict();
	}
  // 加载1个字典
  private void loadDictFile(DictSegment dict, Path file, boolean critical, String name) {
		try (InputStream is = new FileInputStream(file.toFile())) {
			BufferedReader br = new BufferedReader(
					new InputStreamReader(is, "UTF-8"), 512);
			String word = br.readLine();
			if (word != null) {
				if (word.startsWith("\uFEFF"))// 非法字符
					word = word.substring(1);// 丢弃
				for (; word != null; word = br.readLine()) {
					// 按行循环取
					word = word.trim();//去空格
					if (word.isEmpty()) continue;
					dict.fillSegment(word.toCharArray());// 词加入字典，传参是charArray
				}
			}
		} catch (FileNotFoundException e) {
			logger.error("ik-analyzer: " + name + " not found", e);
			if (critical) throw new RuntimeException("ik-analyzer: " + name + " not found!!!", e);
		} catch (IOException e) {
			logger.error("ik-analyzer: " + name + " loading failed", e);
		}
	}
}
```

可以看出Dictionary是个单例，主要是加载出3个DictSegment分别是主字典、量词字典和停用词字典；加载IK分词器的配置文件`IKAnalyzer.cfg.xml`，该配置文件主要配置了动态字典的位置。初始化时还加载了姓、后缀、介词词典，但是并没有直接用，而是在构建DictSegment时将这些字符加到了公用字典表里，这个后面会介绍。构建DictSegmen的方式是加载1个字典文件遍历文件每一行（每行是1个词），用这个词调用DictSegment的`fillSegment(char[] charArray)`方法。 

# DictSegment

这个类是IK分词器最重要的抽象，它的数组织方式及查询方式决定了中文分词的效率。

