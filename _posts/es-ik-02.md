---
title: elasticsearch-ik插件学习（中）
date: 2021/01/21 16:26:10
categories:
- elasticsearch  

tags:
- elasticsearch
- es-plugin 

keywords:
- elasticsearch, plugin, ik, 插件, IKSegmenter, AnalyzeContext, LetterSegmenter, IKArbitrator
---
# 概述
本系列主要从从源码的角度介绍es-IK插件，本文主要从源码层面介绍ik分词的主线逻辑暂不涉及中文分词的逻辑。

<!-- more -->

# IKSegmenter

IKTokenizer中实现分词功能的类，从这个类开始所有主要概念都是IK自身的了与es关系已不是太大。直接放源码：

```java
public final class IKSegmenter {
	// 字符串reader ，要分的1句话
	private Reader input;
	// 分词器上下文
	private AnalyzeContext context;
	// 分词处理器列表
	private List<ISegmenter> segmenters;
	// 分词歧义裁决器
	private IKArbitrator arbitrator;
  // 配置： 是否只能分词 是否启用远程字典加载  是否启用小写处理 es的env(各种文件路径)
  private  Configuration configuration;
	/**
	 * IK分词器构造函数
	 * @param input
     */
	public IKSegmenter(Reader input ,Configuration configuration){
		this.input = input;
    this.configuration = configuration;
    this.init();
	}
	/**
	 * 初始化
	 */
	private void init(){
		//初始化分词上下文
		this.context = new AnalyzeContext(configuration);
		//加载子分词器
		this.segmenters = this.loadSegmenters();
		//加载歧义裁决器
		this.arbitrator = new IKArbitrator();
	}
  	/**
     * 重置分词器到初始状态
     * @param input
     */
	public synchronized void reset(Reader input) {
    // 主要是重新设置 input 即 要分的词
		this.input = input;
    // 重置分词器上下文
		context.reset();
    // 重置所有子分词器
		for(ISegmenter segmenter : segmenters){
			segmenter.reset();
		}
	}
	/**
	 * 初始化词典，加载子分词器实现
	 * @return List<ISegmenter>
	 */
	private List<ISegmenter> loadSegmenters(){
		List<ISegmenter> segmenters = new ArrayList<ISegmenter>(4);
		//处理字母的子分词器
		segmenters.add(new LetterSegmenter()); 
		//处理中文数量词的子分词器
		segmenters.add(new CN_QuantifierSegmenter());
		//处理中文词的子分词器
		segmenters.add(new CJKSegmenter());
		return segmenters;
	}
	/**
	 * 分词，获取下一个词元
	 * @return Lexeme 词元对象
	 * @throws java.io.IOException
	 */
	public synchronized Lexeme next()throws IOException{
		Lexeme l = null;
		while((l = context.getNextLexeme()) == null ){
			/*
			 * 从reader中读取数据，填充buffer
			 * 如果reader是分次读入buffer的，那么buffer要进行移位处理
			 * 移位处理上次读入的但未处理的数据
			 */
			int available = context.fillBuffer(this.input);
			if(available <= 0){
				//reader已经读完
        // 此处单独对上下文reset 感觉和 reset方法有重复
				context.reset();
				return null;
			}else{
				//初始化指针
				context.initCursor();
				do{
        			//遍历子分词器
        			for(ISegmenter segmenter : segmenters){
        				segmenter.analyze(context);
        			}
        			//字符缓冲区接近读完，需要读入新的字符
        			if(context.needRefillBuffer()){
        				break;
        			}
   				//向前移动指针
				}while(context.moveCursor());
				//重置子分词器，为下轮循环进行初始化
				for(ISegmenter segmenter : segmenters){
					segmenter.reset();
				}
			}
			//对分词进行歧义处理
			this.arbitrator.process(context, configuration.isUseSmart());
			//将分词结果输出到结果集，并处理未切分的单个CJK字符
			context.outputToResult();
			//记录本次分词的缓冲区位移
			context.markBufferOffset();			
		}
		return l;
	}
}
```

从代码中可以看出来在ik整体的分词逻辑中有以下几个重要概念：

1. Lexeme，1个词元，是从输入中分出的1个词， 词元包括以下重要属性：

   1. offset，词元起始位移，是在某个segmentBuff中的起始位置
   2. begin，词元的相对起始位置（  词元在文本中的起始位置=offset + begin）
   3. length，词元长度
   4. lexemeText，词元文本
   5. lexemeType，词元类型

   注意词元重写了equals，hashCode，compareTo方法，其中

   compareTo规定，首字母位置越小且长度越长的词元越小

   equals规定起始位置偏移、起始位置、终止位置相同即为相同词元

   hashCode的计算方式与绝对的起止位置以及长度有关

2. ISegmenter，ik子分词器，分词是由多个子分词器共同完成，Ik子分词器有：

   1. LetterSegmenter，处理字母的自分词器
   2. CN_QuantifierSegmenter，处理中文数量词的子分词器
   3. CJKSegmenter，处理中文的子分词器

3. IKArbitrator，分词歧义裁决器，感觉是从多个分词结果中裁决出实际分词结果

4. AnalyzeContext，上下文，分一个句子或一篇文章时，普遍需要多次分词，而上下文是保存中间状态的，它是驱动分一个句子正常分词的关键。

从上面执行过程中可以看出每次分词的核心逻辑是遍历所以子分词器，然后再调用裁决器，接着就可以从上下文中获取下一个词元了。其中不管是子分词器还是裁决器，处理的对象都是上下文AnalyzeContext。每次遍历完一次自分词器后都会操作AnalyzeContext挪动指针。下面我们来看看这个`AnalyzeContext` 遍历一次子分词器主要操作的代码以及一次分词结束后执行的操作。

# AnalyzeContext

从IKSegmenter的代码中我们可以看出，获取1个词元，上下文进行的主要操作是：

1.  context.fillBuffer(this.input)// 将contextBuffer中的数据填充满
2.  context.initCursor(); // 初始化指针
3. 各个子插件对AnalyzeContext进行处理
4. context.moveCursor() // 移动指针

其中2、3步循环执行知道当前上下文的buff中没有待处理字符串。下面我们 来看看相关源码

```java
class AnalyzeContext {
    //默认缓冲区大小
    private static final int BUFF_SIZE = 4096;
    //缓冲区耗尽的临界值
    private static final int BUFF_EXHAUST_CRITICAL = 100;
    //字符串读取缓冲
    private char[] segmentBuff;
    //字符类型数组
    private int[] charTypes;
    //记录Reader内已分析的字串总长度
    //在分多段分析词元时，该变量累计当前的segmentBuff相对于reader起始位置的位移
    private int buffOffset;
    //当前缓冲区 已处理的指针
    private int cursor;
    //最近一次读入的, segmentBuff 可处理的字串长度
    private int available;
    //子分词器锁
    //该集合非空，说明有子分词器在占用segmentBuff
    private Set<String> buffLocker;
    //原始分词结果集合，未经歧义处理
    private QuickSortSet orgLexemes;
    //LexemePath位置索引表
    private Map<Integer, LexemePath> pathMap;
    //最终分词结果集
    private LinkedList<Lexeme> results;
    //分词器配置项
    private Configuration cfg;
    
    public AnalyzeContext(Configuration configuration) {
        this.cfg = configuration;
        this.segmentBuff = new char[BUFF_SIZE];
        this.charTypes = new int[BUFF_SIZE];
        this.buffLocker = new HashSet<String>();
        this.orgLexemes = new QuickSortSet();
        this.pathMap = new HashMap<Integer, LexemePath>();
        this.results = new LinkedList<Lexeme>();
    }
    /**
     * 根据context的上下文情况，填充segmentBuff
     * 每次分词时都尝试填充 且填充满（BUFF_SIZE）
     * 即每次分词都保证segmentBuff中有BUFF_SIZE个未分词的字符
     * @param reader
     * @return 返回待分析的（有效的）字串长度
     * @throws java.io.IOException
     */
    int fillBuffer(Reader reader) throws IOException {
        int readCount = 0;
        if (this.buffOffset == 0) {
            //首次读取reader
            readCount = reader.read(segmentBuff);
        } else {
            // 非首次读
            // 上次还没处理完的
            int offset = this.available - this.cursor;
            if (offset > 0) {
                //最近一次读取的 > 最近一次处理的，将未处理的字串拷贝到segmentBuff头部
                // segmentBuff 还没处理的数据 放到 segmentBuff头部
                System.arraycopy(this.segmentBuff, this.cursor, this.segmentBuff, 0, offset);
                // 目前待处理字符在buff中最后的指针位置
                readCount = offset;
            }
            //继续读取reader ，以onceReadIn - onceAnalyzed为起始位置，继续填充segmentBuff剩余的部分
            readCount += reader.read(this.segmentBuff, offset, BUFF_SIZE - offset);
        }
        //记录最后一次从Reader中读入的可用字符长度
        // 除了最后一次可能不等于BUFF_SIZE，中间都是BUFF_SIZE
        this.available = readCount;
        // 重置当前指针
        this.cursor = 0;
        return readCount;
    }
    
    /**
     * 初始化buff指针，处理第一个字符
     * 此方法是如果有待分的词，也是每次都会调用
     */
    void initCursor() {
        // 重复！与fillBuffer
        this.cursor = 0;
        // 将待处理的第一个字符串做 进行字符规格化（全半角，大小写）
        this.segmentBuff[this.cursor] = CharacterUtil.regularize(this.segmentBuff[this.cursor], cfg.isEnableLowercase());
        // 识别第一处理字符的类型  CHAR_ARABIC（数字），CHAR_ENGLISH（字母），CHAR_CHINESE，CHAR_OTHER_CJK（日韩），CHAR_USELESS（不处理）
        this.charTypes[this.cursor] = CharacterUtil.identifyCharType(this.segmentBuff[this.cursor]);
    }
    
    /**
     * 指针+1 ，此步骤发生在 上一个字符 已经经过各个子插件处理后
     * 成功返回 true； 指针已经到了buff尾部，不能前进，返回false
     * 并处理当前字符
     */
    boolean moveCursor() {
        if (this.cursor < this.available - 1) {
            // 处置指针+1  对下一个字符进行--格式化，识别类型
            this.cursor++;
            this.segmentBuff[this.cursor] = CharacterUtil.regularize(this.segmentBuff[this.cursor], cfg.isEnableLowercase());
            this.charTypes[this.cursor] = CharacterUtil.identifyCharType(this.segmentBuff[this.cursor]);
            return true;
        } else {
            return false;
        }
    }
   /**
     * 累计当前的segmentBuff相对于reader起始位置的位移
     */
    void markBufferOffset() {
        this.buffOffset += this.cursor;
    }
}
```

首先是上线文中几个重要的变量:

1. segmentBuff:char[]，保存从es-input中读出的待处理的字符数据
2. cursor:int , 遍历到当前segmentBuff第几位了
3. available: int ，当前segmentBuff可处理的字符串长度
4. charTypes:int[]，与segmentBuff对应，记录每个字符的类型
5. buffOfset:int , 对于要分词的文本来说，本次分词后处理到的位置
6. orgLexemes: QuickSortSet，中间状态的分词结果，没有经过裁决处理，由子分词器生成，QuickSortSet是IK分词器专用的Lexem快速排序集合
7. pathMap: Map<Integer, LexemePath>，LexemePath位置索引表，由起义裁决器使用orgLexemes生成
8. results:LinkedList<Lexeme> ，最终分出的词元的list，由outputToResult使用pathMap生成

几个方法：

1. int fillBuffer(Reader reader)   ，填充segmentBuff，保证每次分词时是未处理的4096个字符。
2. initCursor和moveCursor类似，都是将下一个字符格式化（全半角、大小写）并判断字符类型，不同的是moveCursor如果发现无后续字符则会返回false。
3. outputToResult ，主要是再次遍历本次处理的字符，将分词结果词元放到返回结果results中，主要处理的的CJK字符。源码后续分析
4. markBufferOffset，更新buffOffset即处理到当前read中的位置。

知道了AnalyzeContext遍历的方式，下面我们看看再遍历过程中，主要执行的分词操作是怎样的，下面以简单的`LetterSegmenter`字符串器为切入点，看看ik是如何处理英文字符和阿拉伯数字的。

# LetterSegmenter

在分词遍历中，核心执行的逻辑是`ISegmenter.analyze(AnalyzeContext)`，这里我们看看对于字符和数字ik插件是怎么做的。上源码：

```java
class LetterSegmenter implements ISegmenter {
	
	//子分词器标签
	static final String SEGMENTER_NAME = "LETTER_SEGMENTER";
	//链接符号
	private static final char[] Letter_Connector = new char[]{'#' , '&' , '+' , '-' , '.' , '@' , '_'};
	//数字符号
	private static final char[] Num_Connector = new char[]{',' , '.'};
	/*
	 * 词元的开始位置，
	 * 同时作为子分词器状态标识
	 * 当start > -1 时，标识当前的分词器正在处理字符
	 */
	private int start;
	/*
	 * 记录词元结束位置
	 * end记录的是在词元中最后一个出现的Letter但非Sign_Connector的字符的位置
	 */
	private int end;
	/*
	 * 字母起始位置
	 */
	private int englishStart;
	// 字母结束位置
	private int englishEnd;
	 // 阿拉伯数字起始位置
	private int arabicStart;
	// 阿拉伯数字结束位置
	private int arabicEnd;
	
	LetterSegmenter(){
		Arrays.sort(Letter_Connector);
		Arrays.sort(Num_Connector);
		this.start = -1;
		this.end = -1;
		this.englishStart = -1;
		this.englishEnd = -1;
		this.arabicStart = -1;
		this.arabicEnd = -1;
	}
	// 分词处理，遍历每个字符都会触发该方法
	public void analyze(AnalyzeContext context) {
		boolean bufferLockFlag = false;
		//处理英文字母
		bufferLockFlag = this.processEnglishLetter(context) || bufferLockFlag;
		//处理阿拉伯字母
		bufferLockFlag = this.processArabicLetter(context) || bufferLockFlag;
		//处理混合字母(这个要放最后处理，可以通过QuickSortSet排除重复)
		bufferLockFlag = this.processMixLetter(context) || bufferLockFlag;
		//判断是否锁定缓冲区
    //当前处理的字符 可以分出 字母或数字的词元，则为true
		if(bufferLockFlag){
			context.lockBuffer(SEGMENTER_NAME);
		}else{
			//对缓冲区解锁
			context.unlockBuffer(SEGMENTER_NAME);
		}
	}
	
	/* (non-Javadoc)
	 * @see org.wltea.analyzer.core.ISegmenter#reset()
	 */
	public void reset() {
		this.start = -1;
		this.end = -1;
		this.englishStart = -1;
		this.englishEnd = -1;
		this.arabicStart = -1;
		this.arabicEnd = -1;
	}	
	/**
	 * 处理纯英文字母输出
	 * @param context
	 * @return
	 */
	private boolean processEnglishLetter(AnalyzeContext context){
		boolean needLock = false;
		if(this.englishStart == -1){//当前的分词器尚未开始处理英文字符	
			if(CharacterUtil.CHAR_ENGLISH == context.getCurrentCharType()){
				//记录起始指针的位置,标明分词器进入处理状态
				this.englishStart = context.getCursor();
				this.englishEnd = this.englishStart;
			}
		}else {//当前的分词器正在处理英文字符	
			if(CharacterUtil.CHAR_ENGLISH == context.getCurrentCharType()){
				//记录当前指针位置为结束位置
				this.englishEnd =  context.getCursor();
			}else{
				//遇到非English字符,输出词元
				Lexeme newLexeme = new Lexeme(context.getBufferOffset() , this.englishStart , this.englishEnd - this.englishStart + 1 , Lexeme.TYPE_ENGLISH);
				context.addLexeme(newLexeme);
				this.englishStart = -1;
				this.englishEnd= -1;
			}
		}
		
		//判断缓冲区是否已经读完
		if(context.isBufferConsumed() && (this.englishStart != -1 && this.englishEnd != -1)){
            //缓冲以读完，输出词元
            Lexeme newLexeme = new Lexeme(context.getBufferOffset() , this.englishStart , this.englishEnd - this.englishStart + 1 , Lexeme.TYPE_ENGLISH);
            context.addLexeme(newLexeme);
            this.englishStart = -1;
            this.englishEnd= -1;
		}	
		//判断是否锁定缓冲区
		if(this.englishStart == -1 && this.englishEnd == -1){
			//对缓冲区解锁
			needLock = false;
		}else{
			needLock = true;
		}
		return needLock;			
	}
	
	/**
	 * 处理阿拉伯数字输出
	 * @param context
	 * @return
	 */
	private boolean processArabicLetter(AnalyzeContext context){
    // 整体逻辑和英文字符类似
    // 不同是 数字才走生成词元的逻辑，用的是arabicStart 和 arabicEnd 记录
		…………	
	}	
  /**
	 * 处理数字字母混合输出 
	 * 如：windos2000 | linliangyi2005@gmail.com
	 * @param context
	 * @return
	 */
	private boolean processMixLetter(AnalyzeContext context){
		// 整体逻辑和英文字符类似
    // 不同是 只要是字母或数字都要走生成词元的逻辑，用的是start 和 end 记录
		…………	
	}

	/**
	 * 判断是否是字母连接符号
	 * @param input
	 * @return
	 */
	private boolean isLetterConnector(char input){
		int index = Arrays.binarySearch(Letter_Connector, input);
		return index >= 0;
	}
	
	/**
	 * 判断是否是数字连接符号
	 * @param input
	 * @return
	 */
	private boolean isNumConnector(char input){
		int index = Arrays.binarySearch(Num_Connector, input);
		return index >= 0;
	}
}
```

可以看出在字母数字分词器中，主要功能就是碰到连续的字母、数字或(字母和数字)，就生成词元，放到AnalyzeContext的orgLexemes，原始分词结果的集合中，该集合还需要经过裁决器处理。这些词元中可能包含重复的词元，比如`windos2000`会派生出3个词元，`window`,`2000`,`windos2000`。 如果待分词文本长度大于4096，那么可能出现一个单词背切成2个词元的情况。

# IKArbitrator

在介绍歧义裁决器前我们先再说下存储自分词器生成的原始词元的QuickSortSet，它是个双向链表，各个词元按照在原词中的位置排列，起始位置相同的词，长度越长越靠前。

再说下LexemePath，它继承与QuickSortSet，用来保存一组词元，这些词元是有重叠部分，也就是要判断歧义的。它又重写了compareTo比较算法，该算法在歧义裁决时有重要作用，该比较算法依次依照以下几条做判断（前一条相同才走下一条）：

1. 词元链的有效字符长度越长越靠前
2. 词元越少越靠前
3. 路径长度越长越靠前，大部分情况与1相同，但是1是去掉比如“的，了，着，把，个”等这种词的长度
4. 结尾下标越大越靠前（统计学结论，逆向切分概率高于正向切分，因此位置越靠后的优先）
5. 各个词元长度越平均越靠前
6. 词元位置权重越大越靠前

下面我们看看当segmentBuff处理完后，分词歧义裁决器`IKArbitrator`做了什么工作，源码如下：

```JAVA
class IKArbitrator {
    /**
     * 分词歧义处理, 核心逻辑用 orgLexemes 生成 pathMap
     * @param useSmart
     */
    void process(AnalyzeContext context, boolean useSmart) {
        QuickSortSet orgLexemes = context.getOrgLexemes();
      // poll 头
        Lexeme orgLexeme = orgLexemes.pollFirst();
        LexemePath crossPath = new LexemePath();
        while (orgLexeme != null) {
            if (!crossPath.addCrossLexeme(orgLexeme)) {
                // 找到与crossPath不相交的下一个crossPath
                // 此时代表上一个crossPath不会再有重叠的Lexeme
                if (crossPath.size() == 1 || !useSmart) {
                    //crossPath没有歧义 或者 不做歧义处理
                    //直接输出当前crossPath
                    context.addLexemePath(crossPath);
                } else {
                    //对当前的crossPath进行歧义处理
                    QuickSortSet.Cell headCell = crossPath.getHead();
                    LexemePath judgeResult = this.judge(headCell, crossPath.getPathLength());
                    //输出歧义处理结果judgeResult，放在pathMap中
                    context.addLexemePath(judgeResult);
                }
                //把orgLexeme加入新的crossPath中，准备看它有没有其他重叠值
                crossPath = new LexemePath();
                crossPath.addCrossLexeme(orgLexeme);
            }
            orgLexeme = orgLexemes.pollFirst();
        }
        //处理最后的path
        if (crossPath.size() == 1 || !useSmart) {
            //crossPath没有歧义 或者 不做歧义处理
            //直接输出当前crossPath
            context.addLexemePath(crossPath);
        } else {
            //对当前的crossPath进行歧义处理
            QuickSortSet.Cell headCell = crossPath.getHead();
            LexemePath judgeResult = this.judge(headCell, crossPath.getPathLength());
            //输出歧义处理结果judgeResult
            context.addLexemePath(judgeResult);
        }
    } 
    /**
     * 歧义识别
     * @param lexemeCell 歧义路径链表头
     * @param fullTextLength 歧义路径文本长度
     * @return
     */
    private LexemePath judge(QuickSortSet.Cell lexemeCell, int fullTextLength) {
        //候选路径集合
        TreeSet<LexemePath> pathOptions = new TreeSet<LexemePath>();
        //候选结果路径
        LexemePath option = new LexemePath();
        //对crossPath进行一次遍历,同时返回本次遍历中有冲突的Lexeme栈
        Stack<QuickSortSet.Cell> lexemeStack = this.forwardPath(lexemeCell, option);  
        //当前词元链并非最理想的，加入候选路径集合
        pathOptions.add(option.copy());
        //存在歧义词，处理
        QuickSortSet.Cell c = null;
        while (!lexemeStack.isEmpty()) {
            c = lexemeStack.pop();
            //回滚词元链
            this.backPath(c.getLexeme(), option);
            //从歧义词位置开始，递归，生成可选方案
            this.forwardPath(c, option);
            pathOptions.add(option.copy());
        }
        //返回集合中的最优方案
        return pathOptions.first(); 
    }
    /**
     * 向前遍历，添加词元，构造一个无歧义词元组合
     * @return
     */
    private Stack<QuickSortSet.Cell> forwardPath(QuickSortSet.Cell lexemeCell, LexemePath option) {
        //发生冲突的Lexeme栈
        Stack<QuickSortSet.Cell> conflictStack = new Stack<QuickSortSet.Cell>();
        QuickSortSet.Cell c = lexemeCell;
        //迭代遍历Lexeme链表
        while (c != null && c.getLexeme() != null) {
            if (!option.addNotCrossLexeme(c.getLexeme())) {
                //词元交叉，添加失败则加入lexemeStack栈
                conflictStack.push(c);
            }
            c = c.getNext();
        }
        return conflictStack;
    }
    /**
     * 回滚词元链，直到它能够接受指定的词元
     */
    private void backPath(Lexeme l, LexemePath option) {
        while (option.checkCross(l)) {
            option.removeTail();
        }
    }
}
```

IKArbitrator的核心逻辑也比较明显了，就是从orgLexemes找出有相互重叠的词元组成一个LexemePath，这个LexemePath通过歧义裁决器生成1个LexemePath，这个中的词元是相互不重合的。举个例子，比如"中华人民共和国中央人民政府今天成立"这句话的orgLexemes有18个，按排列顺序分别是：中华人民共和国; 中华人民; 中华; 华人; 人民共和国; 人民; 共和国; 共和; 国中; 中央人民政府; 中央; 人民政府; 人民; 民政; 政府; 今天; 天成; 成立; 其中第一个相互重叠的词元组LexemePath是：中华人民共和国; 中华人民; 中华; 华人; 人民共和国; 人民; 共和国; 共和; 国中; 中央人民政府; 中央; 人民政府; 人民; 民政; 政府; 经过歧义裁决器变为的LexemePath仅有2个词元分别是"中华人民共和国"和"中华人民共和国"。至于裁决的方法主要在`judge()`方法中，大家可以自行debug，不过说实话生成中，一般会希望分出来的词越多越好，这样查询命中率会上升。

# 后续

新生产的LexemePath会放在pathMap中，紧接着在`outputToResult`会将这次词转移到最终词元的results中，当再次调用IKSegmenter的next获取词元是就从results中吐出一个词，并为他配置上实际的词元内容`LexemeText`。

注意之前Lexeme对象是没有词元文本LexemeText的，只有在要吐出一个词元时才会设置LexemeText。这点可以从AnalyzeContext::getNextLexeme()中看出来。并且如果是短的文本（小于4096个char），其实第一次调用IKSegmenter::next()时就已经全部分好并存储在上下文的results中了，后续只是装上文本返回return而已。

特别注意其实IK分词器是有个小坑的，就是如果文本长度大于4096个字符，那么分词器会强行分词，导致文本在4096的位置的分词会非常非常奇怪。

# 总结

IKSegmenter的核心分词逻辑就是变量词的每一个字符，遍历时所有子分词器都会独自做分词逻辑，最后如何设置开启了智能分词的逻辑那么会执行一个歧义决断器，将互有重叠的词元进行重新分词，从而保证分出的词相互不重叠，但实际生产过程中不建议开启这种方式。下一篇我们将介绍中文子分词器——CJKSegmenter，他的逻辑直接影响中文分词的效率与效果。