# book-refactoring2

![更新在线书籍](https://github.com/MwumLi/book-refactoring2/workflows/%E6%9B%B4%E6%96%B0%E5%9C%A8%E7%BA%BF%E4%B9%A6%E7%B1%8D/badge.svg?branch=main&event=push)  

《重构 改善既有代码的设计第二版》中文版  

## 资源获取  

* 在线阅读: <https://book-refactoring2.ifmicro.com>  
* 电子书: [pdf](https://book-refactoring2.ifmicro.com/ebook/refactoring2.pdf), [epub](https://book-refactoring2.ifmicro.com/ebook/refactoring2.pdf), [mobi](https://book-refactoring2.ifmicro.com/ebook/refactoring2.pdf)  
## 构建自己的站点/电子书  

### 准备

1. 请保证 **构建环境** 章节的要求  
2. 克隆并安装依赖  

``` bash
$ git clone https://github.com/MwumLi/book-refactoring2.git
$ npm i
```
### 静态站点  

> 运行下面命令前, 请保证 **构建环境** 章节的要求, 并先使用 `npm install` 去初始化依赖  

执行下面命令:  

``` bash
$ npm run build
```

构建后的结果放在 `_book/` 目录下, 你可以用来静态部署  
### 电子书

执行下面命令, 你将得到 `mobi`, `epub` 以及 `pdf` 三种电子书:  

``` bash
$ npm run ebook
```

> 注意: 构建电子书之前需要安装 [calibre](http://www.calibre-ebook.com/download), 这是 gitbook 构建电子书的必须软件。  

## 构建环境

* Node.js ^10.x - ^11.x LTS 版本    
* gitbook ^3.x: 因为要支持中文搜索    

## 贡献

本文还有很多文案不准确或源码格式乱的问题, 如有发现欢迎可以[提 pr](https://github.com/MwumLi/book-refactoring2/pulls) 进行贡献  
## 感谢  

本书源码来自 [NxeedGoto/Refactoring2-zh](https://github.com/NxeedGoto/Refactoring2-zh.git), 由于为了构建电子书籍, 所以改造成了 gitbook 格式。  