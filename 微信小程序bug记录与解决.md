## 微信小程序bug记录

**textarea**

1. textarea在模拟器上没有padding，可是在真机上会自带padding，而且在外部改不了，并且在安卓和IOS上padding还不一样
![开发工具](https://images2018.cnblogs.com/blog/993343/201808/993343-20180815124009675-1578976289.png)
![IOS](https://images2018.cnblogs.com/blog/993343/201808/993343-20180815124110819-1658691924.png)
第一张图是在开发工具上的，第二张图是在IOS真机上的。从上图可以看出来，在开发工具上显示很正常，而且没有padding，可是在真机上左上角就出现了padding，并且无论你在外部对textarea的padding做任何处理，都无法覆盖。
目前有一种解决方式是根据ios和android的不同平台来给teaxarea设置不同的样式。
解决方法：通过wx.getSystemInfo来获取当前设备的平台（IOS or Android),然后根据不同的平台来设置不同的偏移样式来兼容(可以通过margin:负值)。

2. textarea层级最高，而且还没办法使用z-index来修改，使得自制modal等组件出现问题。这个问题同样在开发工具上没问题，在真机上才出现问题。
   目前的解决方法：在modal弹起的时候，将textarea隐藏掉，具体隐藏方法看**3**

3. textarea使用display:none;visibility: hidden; opacity: 0都隐藏不了。
   目前的解决方法：

   ```css
   textarea.hidden {
       position: relative;
       left: -1000%;
   }
   ```

   ```html
   <textarea hidden={{isHide}} />
   ```

   查阅了官方文档后发现，官方提供了一个hidden这个api，我们可以通过这个api来隐藏掉textarea