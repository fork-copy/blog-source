---
cover: http://ok2pw0x6d.bkt.clouddn.com/Fscnu6VP6rpydajNrGw-HcwVJlq0.png
date: 2017-01-23T00:00:00+08:00
title: 图床on七牛，简单好用的图床插件
tags: [图床, 七牛, chrome]
---

最近在使用过程中发现**图床on微博**出了点问题，响应体的JSON解析错误，不用想都知道肯定是微博修改了响应体的数据结构（微博图片上传接口响应体是html tag和json混在一起，十分专业）。简单修复了一下，测试的时候却发现，微博的图片上传接口变得不再稳定了，经常404。看来微博图床是不能用了，正好我早就觉得微博不是个好图床。缺点如下：

1. 经常性的要重新登陆，麻烦死了
2. 无法获取到完整的上传图片列表
3. 无法删除上传的图片
4. 服务状态不可控，指不定什么时候接口就不能用了

<!--more-->

要想对上传的图片拥有完全的控制权，那么图片一定要上传到自己能够控制的地方去。目前国内比较出名的免费存储空间提供商我所知的就是七牛了，简单看了看七牛的文档，做个图床没问题。用户可以创建免费空间，免费空间提供测试域名，限制如下：

1. 单IP每秒限制请求次数10次，大于10次禁止5秒
2. 单URL限速8Mbps

对于一个图床来说，这个限制完全够用了。

图床其实只需要两个核心接口：

1. 图片上传接口
2. 图片获取接口

至于图片删除什么的，当然七牛也提供，我个人觉得一个图床工具没必要这么麻烦了。

关于上传，七牛封装有现成的SDK，比如[JavaScript SDK](http://o9gnz92z5.bkt.clouddn.com/code/v6/sdk/javascript.html)，这个SDK是基于[Plupload](http://www.plupload.com/)做的，十分麻烦，提供了一大堆不需要的功能，我需要的就是简单的一个POST调用。在文档中心里的[API参考](http://o9gnz92z5.bkt.clouddn.com/article/index.html#kodo-api-handbook)找了一下，找到了。

[文件直传API](http://o9gnz92z5.bkt.clouddn.com/code/v6/api/kodo-api/up/upload.html)，接口定义如下。

```bash
POST / HTTP/1.1
Host:           upload.qiniu.com
Content-Type:   multipart/form-data; boundary=<frontier>
# 参数
token: 七牛上传凭证
file: 所要上传的文件
```

可以看到接口十分的简洁干净，一个token，一个file即可。文档中提出`fileBinaryData`这个参数也是必填，经我测试不需要，只要`token`和`file`这两个参数就可以成功上传。

关于上传凭证的生成，七牛的文档说的很清楚，同时还提供了[JSFiddle的在线示例](http://jsfiddle.net/gh/get/extjs/4.2/icattlecoder/jsfiddle/tree/master/uptoken)，真是业界良心。千言万语不如代码来的直接。

这里摘录一下最后我的实现。

```javascript
function genUpToken(accessKey, secretKey, policy) {
  var policyStr = JSON.stringify(policy)
  var encoded = btoa(utf16to8(policyStr))
  var hash = CryptoJS.HmacSHA1(encoded, secretKey) // npm install crypto-js
  var encodedSign = hash.toString(CryptoJS.enc.Base64)
  var uploadToken = accessKey + ":" + safe64(encodedSign) + ":" + encoded
  return uploadToken
}

function utf16to8(str) {
  var out, i, len, c
  out = ""
  len = str.length
  for(i = 0; i < len; i++) {
    c = str.charCodeAt(i)
    if ((c >= 0x0001) && (c <= 0x007F)) {
      out += str.charAt(i)
    } else if (c > 0x07FF) {
      out += String.fromCharCode(0xE0 | ((c >> 12) & 0x0F))
      out += String.fromCharCode(0x80 | ((c >>  6) & 0x3F))
      out += String.fromCharCode(0x80 | ((c >>  0) & 0x3F))
    } else {
      out += String.fromCharCode(0xC0 | ((c >>  6) & 0x1F))
      out += String.fromCharCode(0x80 | ((c >>  0) & 0x3F))
    }
  }
  return out
}

function safe64(base64) {
  base64 = base64.replace(/\+/g, "-")
  base64 = base64.replace(/\//g, "_")
  return base64
}
```

好了，到此上传文件就搞定了。接下来我们看看该怎样获取所有上传的文件。七牛提供了[资源列举](http://o9gnz92z5.bkt.clouddn.com/article/index.html#kodo-api-handbook)这样的一个接口。听着名字应该就是我们要的，接口定义如下。

```bash
POST /list?<listSpec> HTTP/1.1
Host:           rsf.qbox.me
Content-Type:   application/x-www-form-urlencoded
Authorization:  QBox <AccessToken>
# 参数
bucket: 七牛空间
```

这里又需要一个token，放在 `Authorization` Header里面的，叫做[管理凭证](http://o9gnz92z5.bkt.clouddn.com/article/developer/security/access-token.html)。打开文档看了一下，不算麻烦。代码如下。

```javascript
function genManageToken(accessKey, secretKey, pathAndQuery, body) {
  const str = pathAndQuery + "\n" + body
  const hash = CryptoJS.HmacSHA1(str, secretKey)
  const encodedSign = safe64(hash.toString(CryptoJS.enc.Base64))
  return accessKey + ":" + encodedSign
}
```

管理凭证生成好以后，将bucket参数携带上应该就可以了。这里有一个需要注意的地方。按理来说，既然是一个POST请求，参数应该放到请求体里面才对的。文档用了**接口规格**这么模糊的四个字，规格放哪儿你倒是说一说呀。所以我先开始怎么试都不行，结果都是401“授权错误”，很难查找具体的原因。后来把参数放在query中就可以了。最终图片获取的代码如下。

```javascript
function fetch() {
  const path = "/list?bucket=" + getItem("bucket")
  return axios.post("http://rsf.qbox.me" + path, null, {
    headers: {
      Authorization: "QBox " + genManageToken(
        getItem("accessKey"),
        getItem("secretKey"),
        path,
        "",
      ),
    },
  })
}
```

到这里，核心功能就做好了，剩下的就是UI层面的事情，在*图床on微博*的基础上，将历史记录功能优化成了和Unsplash一样的三栏显示。最终效果如下所示，关于插件的安装使用仓库README中有详细说明，[仓库地址](https://github.com/fate-lovely/pic-on-qiniu)。

![](http://ok2pw0x6d.bkt.clouddn.com/FshihH-X8XPLfs-XQgfPYPr3UbO6.gif)

