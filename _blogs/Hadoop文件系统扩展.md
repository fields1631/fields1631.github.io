---
title: "Hadoop文件系统扩展"
collection: blogs
type: "blogs"
permalink: /blogs/Hadoop文件系统扩展
date: 2021-02-20
---

## 为什么要扩展Hadoop文件系统
有使用其他文件系统的需要, 如IPFS, 以解决HDFS存在的Data Injection等问题。

## 如何扩展
Hadoop提供了FileSystem类作为文件系统的基类, 我们可以在这个基础上实现自己的文件系统。

所有文件系统都是通过ServiceLoader进行加载的, 通过修改hadoop-common-project/hadoop-common/src/main/resources/META-INF/services/org.apache.hadoop.fs.FileSystem来加载我们的文件系统。

MapReduce程序是通过Job Commiter来控制的, 通过编写Job Commiter可以使用我们的文件系统。

## IpfsFileSystem