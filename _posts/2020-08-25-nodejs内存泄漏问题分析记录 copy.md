---
layout:     post
title:      nodejs内存泄漏问题分析记录
date:       2020-08-25
author:    	边黎安
header-img: img/tag-bg.jpg
catalog: true
tags:
    - node
    - heapdump
---
---


## 获取堆栈图？
1. v8-prfile 流式输出  transform
2. 直接 require("heapdump") heapdump.writeSnapshot(path.join(__dirname, filename))


### 给你一个堆栈图怎么看?
本质上十一个大json
```
{
  snapshot: {}, //给…拍快照拍快照 快照看 节点和边的关系
  nodes:[],    每个节点
  edges: [],   每个边
  strings: []  它的含义比较简单，其内部实际上保存的是 node 和 edge 的名称。
}

```

其实很简单 node底下哎有个 meta meta底下有个 count 然后edges 顺序排列

然后edge里面有一项 叫 to_node  整个图就出来了

往下走之前 带给大家两个概念 
```
Retained Size=当前对象大小+当前对象可直接或间接引用到的对象的大小总和 换句话说，Retained Size就是当前对象被GC后，从Heap上总共能释放掉的内存

Shallow Size
对象自身占用的内存大小，不包括它引用的对象。

```

### 内存引用关系图转化为支配树

这一步非常重要 可以很清晰的看到内存积累在哪一处



### 具体是实战 
首先用ezmoniter 
看到 heap_total 269 mb
但是 heap_used 231.7mb



###  可疑点占据了当前堆内存的 80%+
```
common/models/Operator.js、common/models/School.js、common/models/User.js、common/models/OperatorMenu.js
```
疑问是 Node.js 模块机制源码的朋友马上可以敏感的发现：Node.js 本身的模块导入机制是有缓存的，怎么会有这么多重复的实例引入？根据这些提示，我们到了产生泄漏点的地方

Node.js 每个 require 的文件实际上生成了一个 Module 类的实例，并且模块第一次 require 的时候会缓存起来

Moudle 类的构造函数如下

```
var moudle = new Moudle(filename, parent)
Module._cache[filename] = moudle

// lib/module.js
function Module(id, parent) {
  this.id = id;
  this.exports = {};
  this.parent = parent;
  updateChildren(parent, this, false);
  this.filename = null;
  this.loaded = false;
  this.children = [];
}


// lib/internal/module.js
function makeRequireFunction() {
  // 我们调用的 require 方法
  function require() {
    //...
  }
  //...
  require.cache = Module._cache;
  return require;
}


```
所以  delete require.cache[缓存文件全路径] 
其实就是 Model.cache 
```
updateChildren(parent, this, false)
// lib/module.js
function updateChildren(parent, child, scan) {
  var children = parent && parent.children;
  if (children && !(scan && children.includes(child)))
    children.push(child);
}
```

###　分析下这里面做了什么：
每次require 的时候 除了自身的不存在时会缓存自身实例的引用到　 Module._cache, 还会将自身的实例的引用放入　children　这个　引用数组里

那么产生泄漏：

第一个　delete requrie.cache 仅仅清除了　 Module._cache 对文件第一次 require 的实例的引用
第二个　require 实例的 children 数组中的缓存的引用依旧存在



### 总结

delete require.cache 这个操作是许多希望做 Node.js 热更新的同学喜欢写的代码，根据上面的分析要完整这个模块引入缓存的完整去除需要所有引用到此模块的 parent.children 数组里面的引用。

删除后　内存耗费稳定在 60M 左右

### 持续关注　https://bianbiandashen.github.io/　个人博客

 

