---
date: 2013-11-21T00:00:00+08:00
title: 关于disqus的作用原理
tags: [disqus]
---
当初用jekyll-bootstrap做博客的时候,里面默认配置了disqus的评论模块.当初懵懵懂懂,去disqus官网按照提示填了字段以后,然后根据jekyll配置文件`_config.yml`里面的字段`short-name`填了一下,然后评论模块就可以用了.我心里一直觉得不能理解,但是心里觉得可能很麻烦,就一直没有去钻研这个事情,今天有空,耐心看了官方文档,一切豁然开朗.

当初对disqus的了解,就是一个**云端评论中心**.可以将你对一篇文章的评论存储在disqus的中心,然后通过js脚本调用这些评论.但是,我困惑的问题就是:**disqus怎样将评论和相应的页面进行绑定?**

<!--more-->

当我在对一篇文章进行评论的时候,相应的评论会被传送到disqus的云端进行存储,但是disqus怎样识别出这个评论是属于这个页面的呢?可以看到,调用disqus的源码很简单:

```javascript
(function() {
  var dsq = document.createElement('script')
  dsq.type = 'text/javascript'
  dsq.async = true
  dsq.src = 'http://' + disqus_shortname + '.disqus.com/embed.js'
  (document.getElementsByTagName('head')[0] || document.getElementsByTagName('body')[0]).appendChild(dsq)
})();
```

其中,short_name这个属性是在disqus配置中心进行配置的,一个short-name对应于一个网址.

![](http://ww2.sinaimg.cn/large/9b85365djw1f23b14utehj20nk055dgd.jpg)

很显然,short-name这个属性不能用来标识页面,每个页面这个属性都是这样的.那到底是什么在唯一标识页面?

我在disqus的**Home>Developers>JavaScript configuration variables**这篇文章里面找到了如下的内容:

**disqus_shortname**
>Tells the Disqus service your forum's shortname, which is the unique identifier for your website as registered on Disqus. If undefined, the Disqus embed will not load.

**disqus_identifier**
>Tells the Disqus service how to identify the current page. When the Disqus embed is loaded, the identifier is used to look up the correct thread. If disqus_identifier is undefined, the page's URL will be used. The URL can be unreliable, such as when renaming an article slug or changing domains, so we recommend using your own unique way of identifying a thread.

然后我终于明白了.其实不是没有配置,只是jekyll-bootstrap的文件里面没有列出这几个变量.

上面说的很清楚:

- 首先,`disqus_shortname`用来表示一个网站,如果没有这个属性,disqus是无法加载的.这个属性是所有disqus用户都是唯一的.即你无法创建一个和他人相同的disqus_shortname,因为他用来唯一标识你的网站.

- 其次,`disqus_identifier`这个js变量用来唯一表示当前页面,或者称为`thread`.如果这个属性不进行设置的话,页面的URl将用来唯一标识这个页面,所有的评论将会与页面的URL进行绑定.

现在就很清楚了,当你在一个页面上写评论的时候,disqus会将评论存储在你加载disqus所设置的仓库里面,也就是`disqus_shortname`里面,并且与当前页面的URL或者你设置的`disqus_identifier`进行绑定.当你下一次进入这个页面的时候,disqus首先进入仓库,然后根据你这个的identifier寻找所对应的评论.

那么,很明显,用URL是非常不稳定的.因为如果你一旦对页面改名字或者更改了路径等等等会修改url的操作的话,那么所对应的评论就没有了.所以推荐每个页面都设置一下`disqus_identifier`进行唯一标识.
