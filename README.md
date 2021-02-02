
Windows下安装Hugo
　　二进制安装、在官网上下载zip包：https://gohugo.io/getting-started/installing/

解压到你熟悉的盘中 比如：D:\Hugo\bin中    再在你的用户环境变量中的 Path 添加 地址 D:\Hugo\bin


<!-- 在cmd中出现以下信息说明安装成功，没有的话检查下相关路径是否正确。

运行步骤
　　生成站点：win10中在你想要的目录下打开 Power Shell（打开方式按住 shift 鼠标左键点击文件夹）  

1
hugo new site myblog -->
　　

 

 <!-- Hugo官方解说是：https://gohugo.io/getting-started/directory-structure/ -->

运行运行

 hugo server --minify --theme mogege

 打包打包

 hugo


有个 public 放到你的站点目录就可以了


# 语法

- [x] @mentions, #refs, [links](), **formatting**, and <del>tags</del> supported
- [x] list syntax required (any unordered or ordered list supported)
- [x] this is a complete item
- [ ] this is an incomplete item

*This text will be italic*
_This will also be italic_

**This text will be bold**
__This will also be bold__

_You **can** combine them_

* Item 1
* Item 2
  * Item 2a
  * Item 2b

1. Item 1
1. Item 2
1. Item 3
   1. Item 3a
   1. Item 3b

![GitHub Logo](/images/logo.png)
Format: ![Alt Text](url)

As Kanye West said:

> We're living the future so
> the present is our past.


I think you should use an
`<addr>` element here instead.

```javascript
function fancyAlert(arg) {
  if(arg) {
    $.facebox({div:'#foo'})
  }
}
```

def foo():
    if not bar:
        return True

First Header | Second Header
------------ | -------------
Content from cell 1 | Content from cell 2
Content in the first column | Content in the second column

~~this~~