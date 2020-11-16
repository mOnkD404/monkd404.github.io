---
title: '体验原生安卓safebrowsing:AOSP刷入GMS记录'
date: 2020-11-10 11:51:38
tags:
---

## 背景
最近做safebrowsing时很多体验需要参照gms服务的体验，官方镜像可以体验，但是官方是release系统。aosp正常编译是不带gms的，而且webview的开发经常需要和系统webview作对比，因此需要一个能任意替换系统webview的userdebug系统并且带上gms。
需求如下：
1. userdebug系统
  1. 支持随意替换系统webview
  2. 支持adb root，可以查看所有进程，跑chromium测试
2. 刷入gms，带google服务框架，体验完整原生安卓

## 解决方案
最简单的方法是，使用第三方rom（最好修改较小，接近原生），例如魔趣、lineageOS、Omni等热门第三方rom，在recovery中依次刷入rom和opengapps的包即可。
不过我这次用的pixel4 其他第三方系统都不支持，只有lineageOS可以刷，但是刷入之后发现虽然是userdebug的系统，但是却换不了系统webview，怀疑他的编译选项有问题。
因此选择了下面的方法，自己编译aosp，编译时打入opengapps。
----
## 正文开始。 pixel 4 编译aosp+opengapps ，步骤+踩坑汇总：
1. 拉取android 对应分支源码
例如pixel4
```
repo init -u https://android.googlesource.com/platform/manifest -b android-10.0.0_r41
```
2. 下载厂商驱动
  1. 例如pixel4 下载https://developers.google.com/android/drivers#flameqq3a.200805.001
  2. 解压后在源码目录下执行对应脚本，中间需要输入I ACCEPT
3. 拉取opengapps 源码 https://github.com/opengapps/aosp_build
4. 参照opengapps说明操作，其中最后一步拉代码 repo forall -c git lfs pull 非常容易失败要多来几次，失败的话编译会出错
  1. 小tips 建议选择小包，很多Google全家桶apk可以在playstore下载，而且有些和aosp兼容有问题，比如pixel4刷stock包刷完全屏手势失灵，刷nano/super包正常。
  2. non-stock 包想要能自己随意替换系统webview的话需要开启GAPPS_FORCE_WEBVIEW_OVERRIDES := true
  3. 实际就是在配置 frameworks/base/core/res/res/xml/config_webview_packages.xml 这个文件
5. 修改一处代码，否则pixellauncher起不来， 参考 https://c55jeremy-tech.blogspot.com/2019/04/aosppixel-2-romrom.html
```
--- a/services/core/java/com/android/server/wm/WindowManagerService.java
+++ b/services/core/java/com/android/server/wm/WindowManagerService.java
@@ -6001,8 +6001,8 @@ public class WindowManagerService extends IWindowManager.Stub

     @Override
     public void setShelfHeight(boolean visible, int shelfHeight) {
-        mAmInternal.enforceCallerIsRecentsOrHasPermission(android.Manifest.permission.STATUS_BAR,
-                "setShelfHeight()");
+        //mAmInternal.enforceCallerIsRecentsOrHasPermission(android.Manifest.permission.STATUS_BAR,
+        //        "setShelfHeight()");
         synchronized (mWindowMap) {
             getDefaultDisplayContentLocked().getPinnedStackController().setAdjustedForShelf(visible,
                     shelfHeight)
```
6. 常规编译源码
lunch aosp_flame-userdebug
make -j8
如果这步编译有错误，就是上面repo拉apk没拉全，要删了对应目录重来
7. 刷入 fastboot flashall -w
8. 启动之后报设备未获得play保护机制认证，在https://www.google.com/android/uncertified/ 注册设备的id，注册之后过一会就能正常用了。获取设备id的方法 :
```
adb root
adb shell
sqlite3 /data/data/com.google.android.gsf/databases/gservices.db  'select * from main where name = "android_id";'
```
9. android 10.0 开启手势
```
adb shell cmd overlay enable com.android.internal.systemui.navbar.gestural
```

10. 系统webview 替换姿势
  1.  卸载系统webview 用chromium下脚本 android_webview/tools/remove_preinstalled_webview.py
  2. 安装新webview adb install 即可
  3. 参考文档https://chromium.googlesource.com/chromium/src/+/HEAD/android_webview/docs/aosp-system-integration.md
11. 验证结果
  1. 使用系统webview 访问 https://wutong-apk.cdn.bcebos.com/，出现红色风险提示页面，证明系统wbeview+gms工作正常。
  2. 安装chromium源码编译出的自编译包，在设置/开发者选项中，能同时看到：
    1. 官方的系统webview包，com.google.android.webview可以在play store升级
    2. 自编译webview包，com.android.webview
