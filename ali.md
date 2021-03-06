---
title: ali
date: 2019-09-04 11:21:35
tags:
    - 面试
encrypt: true
enc_pwd: ztp1017
---


今年暑假，在阿里巴巴-新零售事业群-供应链平台事业部实习了2个半月的时间。作为后端实习开发，前期主要学习阿里中间件（pandora,hsf,metaq）; 之后投身于到“服务化升级”项目中，设计并实现了“规则转化模块”，参与了“dstunnel下载组件”的升级改造调研。

# 服务化项目的规则转化模块
- 目的：利用配置化替代手工修改代码：
    - 旧版：开发同学 手工添加数据库字段，修改代码实体类定义，重写sql，重跑ODPS生成数据
    - 新版：业务同学 使用配置化的方式，设置规则添加字段，点击运行，前端页面显示
- 挑战：
    - 怎么将前端的复杂配置，抽象为代码中可以保存和传递的数据模型；
    - 怎么将模型转化为可以在ODPS平台中可以运行的sql语句。（类似于hive）
- 步骤：
    1. 细化需求，提取相似的结构，抽象成为领域模型（一个结构表示一个同维数据，跨纬数据需要group by来聚合）

    2. 分析sql的语法规律，尝试用统一的方式来生成sql： 
    
    ![SQL](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190904104752.png)
    3. 解决了一个 循环依赖的问题

# Dstunnel组件优化设计
- 问题：下载较慢
- 通过阅读源码，发现项目中存在的问题
![](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190904105321.png)
- 问题;
    - 泛化调用性能较弱。（原因是基础域的公共服务，没有引入其他小组的数据接口的二方包）
    - metaQ的“串行化”处理可以转为“并行化”处理，先对数据做分片，然后多线程消费。
![](https://selfstudy.oss-cn-beijing.aliyuncs.com/blog/20190904111446.png)
