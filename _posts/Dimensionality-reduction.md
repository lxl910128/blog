---
title: 降维算法介绍
categories:
- ghost in the shell
tags:
- 特征工程
---
# 简介
特征工程直接决定机器学习的质量。在采用独热编码后，特征会成指数型增长，比如做文本分类时每个词都是1个特征，这样特征通常会有上万个，这时就需要使用降维算法减少特征个数，加快学习效率。降维算法主要可以分为：特征抽取和特征选择。  

<!--more-->
**特征抽取**的思想是通过特征之间的关系，组合不同的特征得到新的特征，这样就改变了原始的特征空间，构成了新的特征。而新的特征更具有代表性，并消耗更少的计算机资源。  
主要方法有:
- 主成份分析（PCA）
- 奇异值分解
- Sammon映射

**特征选择**的主要思想是在原有的特征集合中选出一个更具代表性的子集。
主要方法有：
- 卡方检验 对特征进行打分选择排名靠前的特征
- 信息增益 对特征进行打分选择排名靠前的特征
- 递归特征消除算法 将子集的选择看做是一个搜索优化的问题，通过启发式的搜索优化算法来解决
- 岭回归 确定模型的过程中，挑选出那些对模型的训练有重要意义的属性
