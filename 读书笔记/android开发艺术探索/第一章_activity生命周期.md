# activity生命周期全面分析 #
## 典型情况下的生命周期分析 ##
- onCreate :activity正在被创建
- onRestart : activity正在重新启动，当activity从不可见重新变成可见时调用。
- onStart : activity正在启动，可见但不在前台
- onResume ：activity 获得焦点，已经可见并在前台开始活动
- onPause ：onPause的调用是“Another activity comes in front of the activity”，即另一个activity跑到前台来的时候，前一个activity的onPause方法会被调用 ，新activity的onresume需要栈顶activity的onpause的调用，所以在onPause中不能做耗时的操作。
- onStop ：“The activity is no longer visible”，也就是完全不可见的时候调用的，这个完全不可见真的就是指视觉上的完全看不到而已，无论是按home键返回桌面，还是启动另一activity把原activity完全遮住，都会调用onStop。但是当启动的activity是透明的时候，原activity只会进入onPause状态，而不会走到onStop状态，因为原acitivity还是可见的，虽然逻辑上被遮住了，但是视觉上确实是可见的，这一点要注意。
- onDestroy：activity即将被销毁，通常在此做资源回收的工作。
## 异常情况下的生命周期分析 ##
### 导致activity重新创建的情况
- 有系统相关的资源配置改变导致activity被杀死并重新创建。比如屏幕旋转导致activity重新创建。对于这种情况可以在activity中配置configchange来阻止activity重新创建。  
`
android:configChange = "orientation"
`  
- 资源内存不足导致低优先级的activity被杀死。
### 异常情况下恢复activity的数据
- 在异常关闭activity时会在onStop之前调用onSaveInstanceState 方法保持当前activity的状态
- 重新创建之后会把onSaveInstanceState 保存的Bundle对象同时传递给onCreate和onRestoreInstanceState方法，在此作恢复数据的操作，onRestoreInstanceState调用在onStart之后。
# activity的启动模式 #


