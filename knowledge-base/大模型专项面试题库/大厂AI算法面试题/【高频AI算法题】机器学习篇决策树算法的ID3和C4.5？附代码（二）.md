---
title: 【高频AI算法题】机器学习篇决策树算法的ID3和C4.5？附代码（二）
topic: 大模型专项面试题库
sub_topic: 大厂AI算法面试题
doc_type: pdf
source_url: ''
source_path: 大模型专项面试题库/大厂AI算法面试题/【高频AI算法题】机器学习篇决策树算法的ID3和C4.5？附代码（二）.pdf
questions: 0
chars: 1676
build_date: '2026-11-20'
pages: 2
---

# 【高频AI算法题】机器学习篇决策树算法的ID3和C4.5？附代码（二）

> 主题：大模型专项面试题库 › 大厂AI算法面试题　｜　原始文件：大模型专项面试题库/大厂AI算法面试题/【高频AI算法题】机器学习篇决策树算法的ID3和C4.5？附代码（二）.pdf

本文讨论了决策树算法中 ID3 和 C4.5 的相关内容，包括算法详解、两者比较以及 C4.5 的 Python 实现示例等。关键要点包括：

决策树算法简介：决策树是基于树形结构的分类算法，通过选择不同特征作为“节点”对样本分类，ID3 和 C4.5 是其经典变种。

ID3 算法特点：通过信息增益选择最佳特征，简单易懂，但存在偏向选择取值多特征的问题。

C4.5 算法改进：采用增益率作为特征选择标准，支持连续特征和缺失值，通过剪枝技术减少过拟合，实用性更强。

Python 实现 C4.5 示例：虽 scikit - learn 未直接实现 C4.5，但可用 DecisionTreeClassifier 训练决策树，以鸢尾花数据集为例展示了训练、预测、计算精度和打印决策树的过程。

算法意义：ID3 和 C4.5 为后续决策树算法（如 CART、C5.0 等）奠定了基础。

决策树算法详解：ID3 和 C4.5

## 决策树算法 详解：ID3 和 C4.5

决策树是一种基于树形结构的分类算法， 用于根据特征的属性值对样本进行分类。决策树的构建是通过选择不同特征作为“节点”来完成的，每个节点表示一个特征，每个分支代表该特征的一个值，最终的叶子节点代表分类结果 。ID3（Iterative Dichotomiser 3）和C4.5是决策树算法的两种经典变种，它们在构建决策树的过程中有一些不同的选择标准和方法。

## ID3与C4.5的比较Python实现（以C4.5为例）

虽然scikit-learn中并没有直接实现C4.5算法，但它实现了一个决策树算法——DecisionTreeClassifier，其背后的实现是CART（Classification And Regression Trees）。

下面是一个简单的例子，展示如何使用scikit-learn来训练决策树：

代码块from sklearn.tree import DecisionTreeClassifierfrom sklearn.model_selection import train_test_splitfrom sklearn.datasets import load_irisfrom sklearn.metrics import accuracy_scorefrom sklearn.tree import export_text

# 加载鸢尾花数据集data = load_iris()

X = data.datay = data.target

# 拆分数据集为训练集和测试集X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3,random_state=42)

# 创建决策树模型model = DecisionTreeClassifier(criterion='entropy')  # 使用信息增益作为分裂准则（ID3）

model.fit(X_train, y_train)

# 预测y_pred = model.predict(X_test)

# 计算精度accuracy = accuracy_score(y_test, y_pred)

print(f"模型精度：{accuracy:.2f}")

# 打印决策树tree_rules = export_text(model, feature_names=data.feature_names)

print(tree_rules)

## 总结

ID3算法通过信息增益来选择最佳特征，虽然简单易懂，但它存在偏向选择取值多特征的问题。

C4.5改进了ID3算法，采用增益率作为特征选择标准，支持连续特征和缺失值，并通过剪枝技术减少过拟合，具有更强的实用性。

ID3和C4.5都是经典的决策树算法，它们为后续的决策树算法（如CART、C5.0等）奠定了基础。
