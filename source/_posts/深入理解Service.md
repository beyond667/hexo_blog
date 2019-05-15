---
title: 深入理解Service
date: 2019-05-13 22:46:37
tags: Android
---

### 前言
我们都知道，Service服务是Android的四大组件，有启动和绑定两种启动方式.  
如果有人问你,  
先启动后绑定，Service的生命周期会是怎样,先绑定后启动呢?  
onStartCommand返回值有什么区别？  
如何在Service中弹出Dialog?  
IntentService和Service原理？  
Service与进程间又是怎么通信的（IPC：Inter-Process Communication）？  
是不是懵逼了？

### Service基础
Service是一种可以在后台执行长时间运行操作的而没有用户界面的应用组件。  
- 相对于Activity这种在前台运行有用户界面的组件，Service非常适用于去执行那些不需要和用户交互而且还要求长期运行的任务。
- 跟Activity一样都是运行在ui线程（main thread）中，所以不能执行耗时操作(网络请求,拷贝数据库,大文件)。
- Activity耗时操作5秒，Broadcase耗时操作10秒，Service耗时操作20秒，会报ANR异常。
- 如果要执行耗时操作，可以用new Thread或者AsyncTask在子线程中做，也可以在manifest中设置运行在其他进程，或者用IntentService处理耗时操作。
```
// 服务分为远程服务和本地服务。
// android:process=":remote" 即声明为远程服务，不加则为本地服务。
 <service
 android:name="com.baidu.location.f"
 android:enabled="true"
 android:process=":remote" >
 </service>
```

#### 两种启动方式
##### 启动服务
- 通过startService启动服务，服务处于启动状态。
- 会调用Service的onCreate和onStartCommand方法，这时Service与启动组件没有关系，服务会无限期的运行在后台，除非用户执行StopSelf或者stopService，另外系统在内存不足的时候也会回收。回收时会调用onDestory方法。
- 服务处于启动状态时，多次调用startService只会在第一次执行onCreate，但是每次都会执行onStartCommand。
- 我们可以在onStartCommand中接收组件传过来的Intent，但是却不能回调组件告知其处理结果，所以并不能直接与组件交互。但是可以通过在service中发送广播，在组件中接收的方式来处理service的处理结果。
##### 绑定服务
- 通过bindService绑定服务，服务处于绑定状态。  
- 会调用Service的onCreate和onBind方法，这时Service与组件处于绑定状态，组件可以通过绑定服务时创建的ServiceConnection接口直接与服务交互。
- 如果绑定的组件销毁，服务也会销毁。执行onUnbind方法和onDestory方法。组件也可以通过unbindService来解绑。
![Service生命周期](/images/3.1.png)

##### 对比

#### onStartCommand返回值

#### 如何在Service中启动Dialog

#### Service和IntentService原理

#### Service与进程间通信


### 使用场景

#### 进程保活







