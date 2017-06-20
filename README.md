# js-sdk


## 需求，微信公众号网页开发，如何将选择图库或者调用摄像头后获取的图片进行base64转码。

微信的js-sdk选择的照片会返回一个localIds的数组，每一元素代表了一张图片，可以作为图片的src进行显示:
```
wx.chooseImage({
    count: 1, // 默认9
    sizeType: ['original', 'compressed'], // 可以指定是原图还是压缩图，默认二者都有
    sourceType: ['album', 'camera'], // 可以指定来源是相册还是相机，默认二者都有
    success: function (res) {
        var localIds = res.localIds; // 返回选定照片的本地ID列表，localId可以作为img标签的src属性显示图片
    }
});
```
但由于此locaId是以weixin://resoureceid/xxxxxxxxxxxx**协议开头的，所以不能使用canvas.toDataURL("image/jpeg")进行转码，此方法要求以http或者dataURL的格式才被支持，所以，这时候我们不得不翻阅sdk找到了下面这个方法
```
获取本地图片接口
wx.getLocalImgData({
    localId: '', // 图片的localID
    success: function (res) {
        var localData = res.localData; // localData是图片的base64数据，可以用img标签显示
    }
});
备注：此接口仅在 iOS WKWebview 下提供，用于兼容 iOS WKWebview 不支持 localId 直接显示图片的问题。具体可参考《iOS网页开发适配指南》
```
此方法有两个弊端：
1、iOS 系统里面得到的数据，类型为 image/jgp，不知道是 bug 还是什么原因，因此需要替换一下：
```
localData = localData.replace('jgp', 'jpeg');
```
2、安卓系统得到的数据，是没有 data:image/jpeg;base64, 前缀的。
因此要做一下优化：
```
  wx.chooseImage({
      count: 1, // 默认9
      sizeType: ['compressed'], // 可以指定是原图还是压缩图，默认二者都有
      sourceType: ['camera'], // 可以指定来源是相册还是相机，默认二者都有
      success: function(res) {
          _this.localIds = res.localIds[0]; // 返回选定照片的本地ID列表，localId可以作为img标签的src属性显示图片
         var imgSrc=res.localIds[0];
            wx.getLocalImgData({
               localId: res.localIds[0],
               success: function (res) {
                 var pre="data:image/jpeg;base64,"
                  if(_this.isIOS){//设备为ios时进行替换
                    imgSrc= res.localData.replace('jgp', 'jpeg');
                  }else{
                    imgSrc=pre+res.localData;//设备为android时加上前缀
                   }
                 
               },
               fail:function(){
               }
             });
      },
      fail:function(){   
      }
 });

```
附上判断设备类型的代码：
```
  function deviceType(){
              var u = navigator.userAgent, app = navigator.appVersion;
              this.isAndroid = u.indexOf('Android') > -1 || u.indexOf('Linux') > -1; //g
              this.isIOS = !!u.match(/\(i[^;]+;( U;)? CPU.+Mac OS X/); //ios终端
   }

```
