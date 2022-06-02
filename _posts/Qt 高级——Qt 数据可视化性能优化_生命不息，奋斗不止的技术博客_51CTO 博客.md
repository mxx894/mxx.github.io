> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [blog.51cto.com](https://blog.51cto.com/quantfabric/2399998)

> Qt 高级——Qt 数据可视化性能优化，Qt 高级——Qt 数据可视化性能优化一、数据可视化简介 1、数据可视化简介数据可视化即采用图形图表等对采集的数据进行展示，可以非常直观的查看传感器采集到的数据。

一、数据可视化简介
---------

### 1、数据可视化简介

数据可视化即采用图形图表等对采集的数据进行展示，可以非常直观的查看传感器采集到的数据。本文将使用 Qt 的标准组件 QTableWidget、标准模型、自定义模型分别实现对数据的表格展示。

### 2、系统环境

个人 PC：ThinkPad T450  
操作系统：RHEL7.3 WorkStation  
内存容量：8G  
磁盘容量：SSD 100G  
CPU：Intel® Core™ i5-5200U CPU @ 2.20GHz

二、标准界面组件实现
----------

### 1、代码实现

MainWindow.h 文件：

MainWindow.cpp 文件：

main.cpp 文件：

2、性能分析  
Student 结构体如下：

Student 结构体大小为 108 字节，根据生成的不同数量规模的数据，其程序占用的内存如下：  
![](https://s4.51cto.com//images/blog/201905/25/350452dc3749f5b7981ac330eac1f870.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=)  
根据上述数据，在大规模数据量下，使用 QTableWidget 展示数据时，每条数据实际占用的内存是数据本身大小的 15 倍，数据量越大插入越耗时，头部插入耗时远远大于尾部追加插入。

三、标准模型实现
--------

### 1、代码实现

StudentTableModel.h 文件：

StudentTableModel.cpp 文件：

MainWindow.h 文件：

MainWindow.cpp 文件：

main.cpp 文件：

### 2、性能分析

根据生成的不同数量规模的数据，其程序占用的内存如下：  
![](https://s4.51cto.com//images/blog/201905/25/0040e62effc322601b875b221b98791e.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=)  
使用 QStandardItemModel 与 QTableView 展示数据，每条数据实际占用内存的大小是数据本身大小的 15 倍，数据量越大插入越耗时，头部插入耗时远远大于尾部追加插入，其性能表现与 QTableWidget 相当。

四、自定义模型实现
---------

### 1、代码实现

StudentTableModel.h 文件：

StudentTableModel.cpp 文件：

MainWindow.h 文件：

MainWindow.cpp 文件：

main.cpp 文件：

2、性能分析  
根据生成的不同数量规模的数据，其程序占用的内存如下：  
![](https://s4.51cto.com//images/blog/201905/25/f3bff248daf824df4e1f35f4a4c0c6f2.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=)  
使用 QAbstractTableModel 派生类与 QTableView 展示数据，每条数据实际占用内存的大小是数据本身大小的 1.5 倍，数据量越大插入越耗时，由于底层数据结构采用链表实现，头部插入耗时与尾部追加插入耗时相当，但内存空间占用大幅下降。  
将底层数据结构换成 QVector，根据生成的不同数量规模的数据，其程序占用的内存如下：  
![](https://s4.51cto.com//images/blog/201905/25/4a70e5c52b961d62ee4f22f7ac916223.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=)  
使用 QVector 作为模型的底层数据结构存储数据，其内存占用与 QList 相当，尾部追加插入耗时与 QList 相当，但头部插入比 QList 耗时较多。