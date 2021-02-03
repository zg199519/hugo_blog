---
title: "WeChat shares the development process - 待定 （未完待续）"
date: 2021-02-02
draft: false
tags: ["WeChat-development"]
categories: ["WeChat"]
# featured_image: https://wujunze.com/blog-images/r/pic/20201027144234.png
---

.....

# 准备

[微信开发文档](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/JS-SDK.html)
* 微信开发者工具  
    [在这里下载](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)
* 认证的微信公众号  
    公众号必须是认证的公众号，如果是开发的话也可以申请个人开发账户用作测试！
* JS接口安全域名  
    先登录微信公众平台进入“公众号设置”的“功能设置”里填写“JS接口安全域名”。  
    备注：登录后可在“开发者中心”查看对应的接口权限。

# 步骤
- 绑定域名  
    先登录微信公众平台进入“公众号设置”的“功能设置”里填写“JS接口安全域名”。  
    备注：登录后可在“开发者中心”查看对应的接口权限。
- 引入JS文件  
    在需要调用JS接口的页面引入如下JS文件，（支持https）：http://res.wx.qq.com/open/js/jweixin-1.6.0.js  
    如需进一步提升服务稳定性，当上述资源不可访问时，可改访问：http://res2.wx.qq.com/open/js/jweixin-1.6.0.js （支持https）。  
    备注：支持使用 AMD/CMD 标准模块加载方法加载
- 通过config接口注入权限验证配置  
    所有需要使用JS-SDK的页面必须先注入配置信息，否则将无法调用（同一个url仅需调用一次，  
    对于变化url的SPA的web app可在每次url变化时进行调用,目前Android微信客户端不支持pushState的H5新特性，  
    所以使用pushState来实现web app的页面会导致签名失败，此问题会在Android6.2中修复）。  
    ```bash
        wx.config({
            debug: true, // 开启调试模式,调用的所有api的返回值会在客户端alert出来，若要查看传入的参数，可以在pc端打开，参数信息会通过log打出，仅在pc端时才会打印。
            appId: '', // 必填，公众号的唯一标识
            timestamp: , // 必填，生成签名的时间戳
            nonceStr: '', // 必填，生成签名的随机串
            signature: '',// 必填，签名
            jsApiList: [] // 必填，需要使用的JS接口列表
        });
    ```
- 通过ready接口处理成功验证
    ```bash
        wx.ready(function(){
            // config信息验证后会执行ready方法，所有接口调用都必须在config接口获得结果之后，config是一个客户端的异步操作，所以如果需要在页面加载时就调用相关接口，则须把相关接口放在ready函数中调用来确保正确执行。对于用户触发时才调用的接口，则可以直接调用，不需要放在ready函数中。
        });
    ```
- 通过error接口处理失败验证
    ```bash
        wx.error(function(res){
            // config信息验证失败会执行error函数，如签名过期导致验证失败，具体错误信息可以打开config的debug模式查看，也可以在返回的res参数中查看，对于SPA可以在这里更新签名。
        });
    ```

# 实现

- 引入js sdk  
    原生的 直接在页面 引入即可 如果是 使用框架 可以 使用 第三方插件集成进来 例如 vue `import wx from 'weixin-js-sdk'`
- 调用接口获取所需要的参数  
```bash
    function share (wxConfig, shareData) {
        wx.config({
            debug: false,
            appId: `${wxConfig.appId}`,
            timestamp: `${wxConfig.timestamp}`,
            nonceStr: `${wxConfig.nonceStr}`,
            signature: `${wxConfig.signature}`,
            jsApiList: [
                'onMenuShareTimeline',
                'onMenuShareAppMessage',
                'onMenuShareQQ',
                'onMenuShareWeibo',
                'chooseImage', 'uploadImage', 'previewImage',
                'getLocation'
            ]
        });
        wx.ready(function() {
            //分享到朋友圈
            wx.onMenuShareTimeline({
                title: shareData.title,
                desc: shareData.desc,
                link: shareData.link,
                imgUrl: shareData.icon,
                success: function () {},
                cancel: function () {}
            });
            //分享给朋友
            wx.onMenuShareAppMessage({
                title: shareData.title,
                desc: shareData.desc,
                link: shareData.link,
                imgUrl: shareData.icon,
                type: '',
                dataUrl: "",
                success: function () {},
                cancel: function () {}
            });
        })
    }
```

_写的不对，还请**批评**。_