---
title: FreqAI 开发者指南
subject: FreqAI 开发文档
subtitle: FreqAI 架构和开发说明
short_title: FreqAI 开发
description: 本文档详细介绍了 FreqAI 的项目架构、数据处理流程和文件结构,为开发者提供了深入了解 FreqAI 内部工作原理的指南。
---

# 开发

## 项目架构

FreqAI 的架构和功能被高度通用化，鼓励开发独特的特性、功能、模型等。

类结构和详细的算法概览如下图所示：

![image](assets/freqai_algorithm-diagram.jpg)

如图所示，FreqAI 由三个不同的对象组成：

* **IFreqaiModel** - 一个持久化的单例对象，包含收集、存储和处理数据、特征工程、训练和推理模型所需的全部逻辑。
* **FreqaiDataKitchen** - 一个非持久化对象，为每个唯一资产/模型唯一创建。除了元数据外，还包含多种数据处理工具。
* **FreqaiDataDrawer** - 一个持久化的单例对象，包含所有历史预测、模型及其保存/加载方法。

有多种内置的[预测模型](freqai-configuration.md#using-different-prediction-models)，它们直接继承自 `IFreqaiModel`。每个模型都可以完全访问 `IFreqaiModel` 的所有方法，因此可以随意重写这些函数。然而，高级用户通常只会重写 `fit()`、`train()`、`predict()` 和 `data_cleaning_train/predict()`。

## 数据处理

FreqAI 旨在以简化后处理和增强崩溃恢复能力（通过自动数据重载）的方式组织模型文件、预测数据和元数据。数据保存在 `user_data_dir/models/` 目录结构下，包含所有与训练和回测相关的数据。`FreqaiDataKitchen()` 严重依赖该文件结构以实现正确的训练和推理，因此不应手动修改。

### 文件结构

文件结构会根据[配置](freqai-configuration.md#setting-up-the-configuration-file)中设置的模型 `identifier` 自动生成。以下结构展示了数据在后处理时的存储位置：

| 结构 | 描述 |
|------|------|
| `config_*.json` | 模型专用配置文件的副本。 |
| `historic_predictions.pkl` | 包含在 live 部署期间 `identifier` 模型生命周期内生成的所有历史预测。`historic_predictions.pkl` 用于在崩溃或配置更改后重新加载模型。始终保留一个备份文件以防主文件损坏。FreqAI **自动**检测损坏并用备份替换损坏的文件。 |
| `pair_dictionary.json` | 包含训练队列以及最近训练模型磁盘位置的文件。 |
| `sub-train-*_TIMESTAMP` | 包含与单个模型相关的所有文件的文件夹，如：<br>
|| `*_metadata.json` - 模型元数据，如归一化最大/最小值、预期训练特征列表等。<br>
|| `*_model.*` - 保存到磁盘的模型文件，用于崩溃恢复。可能为 `joblib`（常见提升库）、`zip`（stable_baselines）、`hd5`（keras 类型）等。<br>
|| `*_pca_object.pkl` - [主成分分析（PCA）](freqai-feature-engineering.md#data-dimensionality-reduction-with-principal-component-analysis) 变换对象（如果配置中设置了 `principal_component_analysis: True`），用于变换未见过的预测特征。<br>
|| `*_svm_model.pkl` - [支持向量机（SVM）](freqai-feature-engineering.md#identifying-outliers-using-a-support-vector-machine-svm) 模型（如果配置中设置了 `use_SVM_to_remove_outliers: True`），用于检测未见过的预测特征中的异常值。<br>
|| `*_trained_df.pkl` - 包含用于训练 `identifier` 模型的所有训练特征的数据框。用于计算 [Dissimilarity Index (DI)](freqai-feature-engineering.md#identifying-outliers-with-the-dissimilarity-index-di)，也可用于后处理。<br>
|| `*_trained_dates.df.pkl` - 与 `trained_df.pkl` 相关的日期，便于后处理。|

示例文件结构如下：

```
├── models
│   └── unique-id
│       ├── config_freqai.example.json
│       ├── historic_predictions.backup.pkl
│       ├── historic_predictions.pkl
│       ├── pair_dictionary.json
│       ├── sub-train-1INCH_1662821319
│       │   ├── cb_1inch_1662821319_metadata.json
│       │   ├── cb_1inch_1662821319_model.joblib
│       │   ├── cb_1inch_1662821319_pca_object.pkl
│       │   ├── cb_1inch_1662821319_svm_model.joblib
│       │   ├── cb_1inch_1662821319_trained_dates_df.pkl
│       │   └── cb_1inch_1662821319_trained_df.pkl
│       ├── sub-train-1INCH_1662821371
│       │   ├── cb_1inch_1662821371_metadata.json
│       │   ├── cb_1inch_1662821371_model.joblib
│       │   ├── cb_1inch_1662821371_pca_object.pkl
│       │   ├── cb_1inch_1662821371_svm_model.joblib
│       │   ├── cb_1inch_1662821371_trained_dates_df.pkl
│       │   └── cb_1inch_1662821371_trained_df.pkl
│       ├── sub-train-ADA_1662821344
│       │   ├── cb_ada_1662821344_metadata.json
│       │   ├── cb_ada_1662821344_model.joblib
│       │   ├── cb_ada_1662821344_pca_object.pkl
│       │   ├── cb_ada_1662821344_svm_model.joblib
│       │   ├── cb_ada_1662821344_trained_dates_df.pkl
│       │   └── cb_ada_1662821344_trained_df.pkl
│       └── sub-train-ADA_1662821399
│           ├── cb_ada_1662821399_metadata.json
│           ├── cb_ada_1662821399_model.joblib
│           ├── cb_ada_1662821399_pca_object.pkl
│           ├── cb_ada_1662821399_svm_model.joblib
│           ├── cb_ada_1662821399_trained_dates_df.pkl
│           └── cb_ada_1662821399_trained_df.pkl

```
