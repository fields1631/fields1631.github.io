---
title: "IPFS with Access Control"
collection: blogs
type: "blogs"
permalink: /blogs/IPFS-with-Access-Control
date: 2021-02-13
---

* IPFS不支持访问控制, 是否可以使用CID来解决这一问题?
  * CID = \<cid-veresion\>\<multicodec\>\<multihash\>
  * 如果使用非对称加密 (如RSA) 对数字签名进行加密, 然后将数字签名, 证书等附在CID中, 使用对称加密对内容加密, 是否可以实现访问控制?

* 存在的问题
  * 如何使用加密来进行访问控制?
  * 这个机制如何在CID中集成?


* 文献阅读
  * **[Cryptographic Access Control in a Distributed File System](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.58.889&rep=rep1&type=pdf)**中说明访问控制需要reference monitor与access control list, 而在分布式系统中这不可能, 并且经常有failure的问题。文章正是提出了不需要reference monitor的机制。
