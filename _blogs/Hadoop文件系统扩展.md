---
title: "Hadoop文件系统扩展"
collection: blogs
type: "blogs"
permalink: /blogs/Hadoop文件系统扩展
date: 2021-02-20
---

本文介绍对Hadoop文件系统的一些思考和研究。

## 为什么要扩展Hadoop文件系统

有使用其他文件系统的需要, 如IPFS, 以解决HDFS存在的Data Injection等问题。

## 如何编写自己的文件系统
Hadoop提供了FileSystem类作为文件系统的基类, 我们可以在这个基础上实现自己的文件系统。目前Hadoop有HttpFileSystem, HarFileSystem等, 可以参考它们的实现。

所有文件系统都是通过ServiceLoader进行加载的, 通过修改hadoop-common-project/hadoop-common/src/main/resources/META-INF/services/org.apache.hadoop.fs.FileSystem来加载我们的文件系统。

MapReduce程序是通过Job Commiter来控制的, 通过编写Job Commiter可以使用我们的文件系统。

## IpfsFileSystem
### 成员
* uri：形式为"ipfs://hash/path/to/file"或"ipfs://hash"。其中ipfs作为uri的scheme；hash为哈希值，作为uri的authority或host；/path/to/file为文件路径，作为uri的path。

* scheme："ipfs"，指定uri的scheme。

## 方法

* FSDataInputStream open(Path path, int i)：判断路径是否存在，如果不存在或是文件夹要报错。创建一个FSDataInputStream的子类IpfsDataInputStream，然后类型转换成FSDataInputStream返回，就可以使用IpfsDataInputStream来读取数据。目前看来应该是可以用子类来代替FSDataInputStream的（例：AbstractHttpFileSystem中的open，返回的是new FSDataInputStream(new HttpDataInputStream(in))，而我们不需要一个AbstractFileSystem，因为它只是为了实现http和https的FileSystem）。ISSUE：是否禁止此操作？在毕设中没有去实现这一方法，应该是trivial的。

* FSDataOutputStream create(Path path, FsPermission fsPermission, boolean b, int i, short i1, long l, Progressable progressable)：根据传入的路径创建文件，如果路径是文件夹要报错。创建一个FSDataOutputStream的子类IpfsDataOutputStream，然后类型转换成FSDataOutputStream返回。实现参考毕设中的思路：

  > The create function is used to create files at a given Path, returning a stream to which file contents are sent. My implementation of the function first performs a check on the type of file Path it is given. This Path must either be a relative string Path (of the form foo/bar) or an IPNS string Path (of the form hash/foo/bar). This is due to the way mutability is handled by our implementation.
  >
  > If the given Path is of the right form, a check is performed to ensure the root hash (either that given in the IPNS string Path or the hash that corresponds to the current working directory) is mutable. If so, the mkdirs function is called, as needed, to construct the Path of the file to be created and finally an FSDataOutputStream is created from a new IPFSOutputStream object, and is returned. The IPFSOutputStream is covered in more depth in the next section, but has the basic functionality of accepting writes to the new File and saving the file contents to IPFS once the stream is closed or flushed.

* FSDataOutputStream append(Path path, int i, Progressable progressable)：判断路径是否存在，如果不存在要报错，如果是文件夹也要报错。然后返回IpfsDataOutputStream。

* void concat(Path trg, Path[] psrcs)：按照普通的文件系统的逻辑来拼接路径，trivial。ISSUE：补充这个逻辑。

* boolean delete(Path f, boolean recursive)：判断路径是否存在，不存在则报错，存在则判断是否是文件夹，并且根据recursive来递归删除。实现参考毕设中的思路：

  > Once it has been determined that the given Path is of the correct format, and that it is mutable, the function checks whether the Path refers to a directory. If so, as long as the recursive flag is true, delete is called recursively on all the children of the current Path.
  >
  > When any recursion has hit the base case, the parent of the given Path is modified to remove the link to the child (which is being deleted). This creates a new hash for the parent, and so the parent of the parent must be modified so that it’s link to the parent points to the new parent (with removed link) and not the old parent. This process of updating links continues up the path to the root object until a new object is created for the root. This new object’s hash is then published to IPNS such that subsequent operations on the working directory will be operating on the now modified working directory.



### 其他类
* IpfsDataInputStream：extends FSDataInputStream，储存的uri形式为"ipfs://hash/path/to/file"，在调用读入的方法时使用IPNS retrieve要读入的文件。

* IpfsDataOutputStream：extends FSDataOutputStream，储存的uri形式为"ipfs://hash/path/to/file"，在调用写入的方法时使用IPNS publish写入的文件。功能有：写入、追加等。因此需要有如下方法：

  ```
  public void close() throws IOException;
  public void flush() throws IOException;
  public void write(int b) throws IOException;
  public void write(byte[] b) throws IOException;
  public void write(byte[] b, int off, int len) throws IOException;
  ```
