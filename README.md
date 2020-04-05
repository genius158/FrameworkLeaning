# FrameworkLeaning
### 编译系统([编译前准备](https://juejin.im/post/5da29dc9f265da5b633cdc8e))
0. [MacOSX10.11.sdk.tar.xz](MacOSX10.11.sdk.tar.xz)解压放到/Applications/XCode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/
1. 在 Mac OS 中，可同时打开的文件描述符的默认数量上限太低，在高度并行的编译流程中，可能会超出此上限。要提高此上限，请将下列行添加到 ~/.bash_profile 中：
ulimit -S -n 1024
2. $ source build/envsetup.sh(使用envsetup.sh脚本初始化环境)
3. $ lunch 
<br/>
user:设定属性ro.secure=1,打开安全检查功能 \ 设定属性ro.debuggable=0,关闭应用调试功能 \ 默认关闭adb功能 \ 打开Proguard混淆器 \ 打开DEXPREOPT预先编译优化	
<br/>
userdebug:设定属性ro.secure=1,打开安全检查功能 \ 设定属性ro.debuggable=1,启用应用调试功能 \ 默认打开adb功能 \ 打开Proguard混淆器 \ 打开DEXPREOPT预先编译优化	
<br/>
eng:设定属性ro.secure=0,关闭安全检查功能 \ 设定属性ro.debuggable=1,启用应用调试功能 \ 默认打开adb功能 \ 关闭Proguard混淆器 \ 关闭DEXPREOPT预先编译优化	\ 设定属性ro.kernel.android.checkjni=1,启用JNI调用检查
<br/>
4. $ sysctl -n machdep.cpu.core_count(查看内核数)
5. $ make -j4（4来自3的数量）

## logic
![app_main](app_main.png) 