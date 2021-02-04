# 一般情况下的生命周期 #
- onCreate：创建，可做一些初始化工作比如：加载界面布局初始化Activity所需数据。
- onRestart:Activity正在重新启动，当前activity从不可见变为可见onReStart就会被调用。
- onStart:表示activity已经被创建并且正在启动，当前是可见状态，但是还在后台，无法进行交互
- onResume: 表示Activity已经可见并且出现在前台，可以进行用户交互。
- onPause:onPause的调用是“Another activity comes in front of the activity”，即另一个activity跑到前台来的时候，前一个activity的onPause方法会被调用。onPause必须执行完，新activity的onResume才会执行。所以此时不宜进行耗时操作。
- onStop：“The activity is no longer visible”，也就是完全不可见的时候调用的。
- onDestroy: 表示activity即将被销毁。可在此处做回收工作以及最终资源的释放。
- onNewIntent：有特殊启动模式如singleTop等，存在activity实例时再打开新activity时不会调用oncerate方法，会走onNewIntent方法。
#异常情况下的生命周期  #


# 相关扩展
## a 启动b activity 的生命周期变化
-   Main.onCreate()---->Main.onStart()---->Main.onResume()-----“点击按钮，开启一个新的Activity”------>Main.onPause()---------> Second.onCreate()----> Second.onStart()----> Second.onResume()---->Main.onStop()-------“到此，MainActivity  启动  SecondActivity 过程中就执行完毕”
## 点击Back按钮 回到前一个Activity  ，生命周期变化
- “点击Back按钮”--------->Second.onPause()---->Main.onReStart()---->Main.onStart()---->Main.onResume()---->Second.onStop()---->Second.onDestory()

## 启动一个dialog，activity的生命周期
- 无变化，除非是启动一个activity的dialog，activity的生命周期相当于activity