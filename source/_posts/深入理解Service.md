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
  
  ##### 既绑定又启动的生命周期
  
  我们用代码来验证下：  
  
  MyService.java
  
  ```
  public class MyService extends Service {
    public MyService() {
    }
  
    @Override
    public void onCreate() {
        super.onCreate();
        Log.e("==", "======onCreate==");
    }
  
    @Override
    public IBinder onBind(Intent intent) {
        Log.e("==", "======onBind==");
        return new MyBinder();
    }
  
    @Override
    public boolean onUnbind(Intent intent) {
        Log.e("==", "======onUnbind==");
        return super.onUnbind(intent);
    }
  
    @Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        Log.e("==", "======onStartCommand==");
        return START_STICKY;
    }
  
    @Override
    public void onDestroy() {
        Log.e("==", "======onDestroy==");
        super.onDestroy();
    }
  
    class MyBinder extends Binder {
        public MyService getService() {
            return MyService.this;
        }
    }
  
    public void myMethod() {
    }
  }
  ```

ServiceActivity.class

```
public class ServiceActivity extends AppCompatActivity{

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_service);


        ButterKnife.bind(this);
    }
    @OnClick({R.id.bt_start,R.id.bt_stop,R.id.bt_bind,R.id.bt_unbind,
            R.id.bt_bind_important,R.id.bt_not_foreground,R.id.bt_debug_unbind})
    void clickView(View view) {
        Intent intent = new Intent(ServiceActivity.this,MyService.class);
        switch (view.getId()) {
            case R.id.bt_start:
                startService(new Intent(ServiceActivity.this,MyService.class));
                break;
            case R.id.bt_stop:
                stopService(new Intent(ServiceActivity.this,MyService.class));
                break;
            case R.id.bt_bind:
                bindService(intent,connection,BIND_AUTO_CREATE);
                break;
            case R.id.bt_unbind:
                unbindService(connection);
                break;
            default:
                break;
        }
    }

    private ServiceConnection connection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName componentName, IBinder iBinder) {
            MyService.MyBinder binder = (MyService.MyBinder)iBinder;
            binder.getService().myMethod();
        }

        @Override
        public void onServiceDisconnected(ComponentName componentName) {
            Log.e("===","=====onServiceDisconnected");
        }
    };
}
```
具体操作步骤(模版插入表格有问题，后面再修)：
| 步骤           | 方法调用顺序 | 结论 |  
| -----------                        | ---------------------------------------------------------------------   | ---------------------------- |   
| start(1),stop(2)                   | onCreate->onStartCommand(1)->onDestory(2)                               | 正常startService              |
| start(1),start(2),stop(3)          | onCreate->onStartCommand(1)->onStartCommand(2)->onDestory(3)            | 多次start只第一次调用onCreate   |
| bind(1),unbind(2)                  | onCreate->onBind(1)->onUnbind->onDestory(2)                             | 正常bindService               |
| bind(1),bind(2),unbind(3)          | onCreate->onBind(1)->null(2)->onUnbind->onDestory(3)                    | 多次bind不会重复调用onBind      |
| start(1),unbind(2),stop(3)         | onCreate->onStartCommand(1)->Service not registered(2)                  | unbind调用时必须bind           |
| bind(1),stop(2),unbind(3)          | onCreate->onBind(1)->null(2)->onUnbind->onDestory(3)                    | 绑定状态通过stop无效            |
| start(1),bind(2),stop(3),unbind(4) | onCreate->onStartCommand(1)->onBind(2)->null(3)->onUnbind->onDestory(4) | 见底                          |
| start(1),bind(2),unbind(3),stop(4) | onCreate->onStartCommand(1)->onBind(2)->onUnbind(3)->onDestory(4)       | 见底                          |
| bind(1),start(2),unbind(3),stop(4) | onCreate->onBind(1)->onStartCommand(2)->onUnbind(3)->onDestory(4)       | 见底                          |
| bind(1),start(2),stop(3),unbind(4) | onCreate->onBind(1)->onStartCommand(2)->null(3)->onUnbind->onDestory(4) | 见底                          | 

我们可以看出规律：  
先启动后绑定和先绑定再启动，都是先onCreate。onStartCommand和onBind再根据启动和绑定调用顺序执行；
先stop不会调用任何方法，后解绑会执行unbind和onDestory；
先unbind只会调onUnbind并不会立即销毁，而是需要后调stop销毁。  
我们注意到，bindService最后一个参数我们一般用BIND_AUTO_CREATE，还有其他标识,例如：BIND_DEBUG_UNBIND，BIND_IMPORTANT，BIND_NOT_FOREGROUND。  
如果service未启动，设置成BIND_AUTO_CREATE绑定service后会立即执行onBind，而其他不会，只有startService之后才会执行onBind。  
这时候还是相当于先绑定后启动，onCreate->onBind->onStartCommand，注意这里不同的是，如果调stop，会直接调onUnbind->onDestory,就不需要再解绑了。  
另外这里还会调用onServiceDisconnected，有没有注意到onServiceDisconnected这个方法不管通过解绑还是stop都没调用到？
对于onServiceDisconnected调用时机，官方原话是这样的:

```
Called when a connection to the Service has been lost. This typically happens when the process hosting the service has crashed or been killed. This does not remove the ServiceConnection itself -- this binding to the service will remain active, and you will receive a call to onServiceConnected(ComponentName, IBinder) when the Service is next running.
```

服务所在的进程如果crash或者被系统杀死会回调，我杀掉app发现根本没有回调这个方法。测试结果也只有在非BIND_AUTO_CREATE绑定时stop后会回调。

#### onStartCommand返回值
Service.class
```
      public @StartResult int onStartCommand(Intent intent, @StartArgFlags int flags, int startId) {
        onStart(intent, startId);
        return mStartCompatibility ? START_STICKY_COMPATIBILITY : START_STICKY;
    }
```
Service默认START_STICKY_COMPATIBILITY或者START_STICKY。另外还有START_NOT_STICKY和START_REDELIVER_INTENT，那么这几个参数有什么区别呢？
- START_STICKY。如果Service所在的进程，在执行了onStartCommand方法后，被清理了，那么这个Service会被保留在已开始的状态，但是不保留传入的Intent，随后系统会尝试重新创建此Service，由于服务状态保留在已开始状态，所以创建服务后一定会调用onStartCommand方法。如果在此期间没有任何启动命令被传递到service，那么参数Intent将为null，需要我们小心处理。
- START_NOT_STICKY。如果Service所在的进程，在执行了onStartCommand方法后，被清理了，则系统不会重新启动此Service。
- START_REDELIVER_INTENT。如果Service所在的进程，在执行了onStartCommand方法后，被清理了，和START_STICKY一样，也会重新创建此Service并调用onStartCommand方法。不同之处在于，重新创建Service时onStartCommand方法会传入之前的intent。  
- START_STICKY_COMPATIBILITY。START_STICKY的兼容版本，但是不能保证被清理后onStartCommand方法一定会被重新调用。

我们注意到，onStartCommand有3个参数，intent是创建的时候传进来的，flags和startId又是什么呢？  

flag有3个值，0，START_FLAG_REDELIVERY和START_FLAG_RETRY。  
- 正常创建Service时onStartCommand传入的flags为0。  
- 如果onStartCommand返回的是START_REDELIVER_INTENT，在service被系统清理掉重建service时，flags就是START_FLAG_REDELIVERY，intent不为null。  
- 如果Service创建过程中，onStartCommand方法未被调用或者没有正常返回的异常情况下， 再次尝试创建，传入的flags就为START_FLAG_RETRY。

startId代表这个唯一的启动请求，每次调用startService这个startId都不同。那这个参数有什么意义呢？  
停止service时有4种方式，stopService，stopSelf(),stopSelf(startId),stopSelfResult(startId),前两个直接停止服务，而后两个只有传入的是最后一个startId才能停止服务。举个例子，用户停止服务的时候又来了一个新的启动请求，1，2会立即停止服务，而3，4由于没有拿到最新的startid是不会停止成功的。3和4的区别是stopSelfResult有返回布尔值。
#### 如何在Service中启动Dialog
Service中可以直接Toast，如果要弹dialog，只能是弹系统dialog。  
Myservice.class
```
  private void showDialog() {
        AlertDialog.Builder builder = new AlertDialog.Builder(this);
        builder.setTitle("提示");
        builder.setMessage("这里是service弹出的系统dialog");
        builder.setNegativeButton("取消", new DialogInterface.OnClickListener() {
            @Override
            public void onClick(DialogInterface dialog, int which) {
            }
        });
        builder.setPositiveButton("确定", new DialogInterface.OnClickListener() {
            @Override
            public void onClick(DialogInterface dialog, int which) {
            }
        });
        final AlertDialog dialog = builder.create();
        //在dialog  show方法之前添加如下代码，表示该dialog是一个系统的dialog
        //8.0新特性
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY);
        } else {
            dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ALERT);
        }
        dialog.show();
    }
```
对于8.0以上的系统，需要申请悬浮框权限
```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (!Settings.canDrawOverlays(ServiceActivity.this)) {
                //若没有权限，提示获取.
                Intent intent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION);
                Toast.makeText(ServiceActivity.this,"需要取得权限以使用悬浮窗",Toast.LENGTH_SHORT).show();
                startActivity(intent);
            }
        }
```
对应的manifest中加入SYSTEM_ALERT_WINDOW权限
```
<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />
```
#### Service和IntentService原理
通过上面的分析，我们知道Service的运行主线程，对于耗时操作可能会引起ANR，所以耗时的操作都需要放在子线程中去做。  
Android也提供了Service的衍生类IntentService来处理耗时操作。  
IntentService保留了Service原有的特性，并且将耗时的操作都放在的子线程中，通过Handler的回调方式实现了数据的返回。  
IntentService源码
```
public abstract class IntentService extends Service {
    //HandlerThread的looper
    private volatile Looper mServiceLooper;
    //通过looper创建的一个Handler对象，用于处理消息
    private volatile ServiceHandler mServiceHandler;
    private String mName;
    private boolean mRedelivery;

    /**
    * 通过HandlerThread线程中的looper对象构造的一个Handler对象
    * 可以看到这个handler中主要就是处理消息
    * 并且将我们的onHandlerIntent方法回调出去
    * 然后停止任务，销毁自己
    */
    private final class ServiceHandler extends Handler {
        public ServiceHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            onHandleIntent((Intent)msg.obj);
            stopSelf(msg.arg1);
        }
    }

    /**
    * 当我们的IntentService第一次启动的时候，onCreate方法会执行一次
    * 可以看到方法里创建了一个HandlerThread
    * HandlerThread继承Thread，它是一种可以使用Handler的Thread
    * 它的run方法里通过Looper.prepare()来创建消息队列
    * 并通过Looper.loop()来开启消息循环，所以就可以在其中创建Handler了
    * 这里我们通过HandlerThread得到一个looper对象
    * 并且使用它的looper对象来构造一个Handler对象，就是我们上面看到的那个
    * 这样做的好处就是通过mServiceHandler发出的消息都是在HandlerThread中执行
    * 所以从这个角度来看，IntentService是可以执行后台任务的
    */
    @Override
    public void onCreate() {
        super.onCreate();
        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
        thread.start();

        mServiceLooper = thread.getLooper();
        mServiceHandler = new ServiceHandler(mServiceLooper);
    }

    /**
    * 通过handler发送了一个消息，把我们的intent对象和startId发送出去
    */
    @Override
    public void onStart(@Nullable Intent intent, int startId) {
        Message msg = mServiceHandler.obtainMessage();
        msg.arg1 = startId;
        msg.obj = intent;
        mServiceHandler.sendMessage(msg);
    }

    /**
    * 每次启动IntentService，onStartCommand()就会被调用一次
    * 在这个方法里处理每个后台任务的intent
    * 可以看到在这个方法里调用的是上方的onStart()方法
    */
    @Override
    public int onStartCommand(@Nullable Intent intent, int flags, int startId) {
        onStart(intent, startId);
        return mRedelivery ? START_REDELIVER_INTENT : START_NOT_STICKY;
    }

    /**
    * 因为looper是无限循环轮询消息的一个机制，所以当我们销毁时需要调用它的quit()方法来终止
    */
    @Override
    public void onDestroy() {
        mServiceLooper.quit();
    } 

    @Override
    @Nullable
    public IBinder onBind(Intent intent) {
        return null;
    }

    /**
    * 这就是IntentService中定义的抽象方法
    * 具体交由它自己的子类来实现
    */ 
    @WorkerThread
    protected abstract void onHandleIntent(@Nullable Intent intent);
}
```
通过源码看到：
1. IntentService继承Service，专门处理异步请求。所以用法跟Service基本一样，但只能通过startService来启动。
2. IntentService的onBind方法返回null，所以如果你通过onBind启动的话相当于绕过IntentService的onStartCommand方法，也就绕过了handler的sendMessage方法，在你自定义的service的onHandleIntent方法就不会被调用，相当于你没用IntentService而是用的普通Service。
3. IntentService本质上是通过HandlerThread，Message，Looper实现的在service中的异步线程消息处理机制。
4. onStart方法在Service的生命周期中不会执行，但是在IntentService中会在onStartCommand后执行，另外在子类的onStartCommand方法中不能直接return START_STICKY等，而是super.onStartCommand。
5. 在所有的looper任务处理结束后会将自身服务关闭，不需要用户手动的停止服务。
具体使用请参照链接：[IntentService使用例子](https://github.com/beyond667/study/blob/master/app/src/main/java/demo/beyond/com/blog/service/MyIntentService.java "IntentService使用例子")

#### Service与进程间通信（IPC）- AIDL
AIDL(Android Interface Definition Language，即Android接口定义语言),用于定义服务器和客户端通信接口的一种描述语言，可以拿来生成用于 IPC 的代码, AIDL这门语言的目的就是为了实现进程间通信，可以在一个进程中获取另一个进程的数据和调用其暴露出来的方法，从而实现进程间通信。
##### AIDL语法
- 服务端定义AIDL文件。在src目录下新建.aidl文件。写完后build后会生成java文件。  
IMyAidlInterface.aidl
```
//即使IMyListener在同一目录下，也需要手动import进来，android studio不会自动引入
import demo.beyond.com.blog.IMyListener;
interface IMyAidlInterface {
    void operation(int parameter1 , int parameter2, IMyListener listener);
}
```
IMyListener.aidl
```
interface IMyListener {
   void onSuccess(in int result);
}

```
AIDL支持的数据类型分为如下几种： 
1 八种基本数据类型：byte、char、short、int、long、float、double、boolean，String，CharSequence  
2 实现了Parcelable接口的数据类型  
3 List 类型。List承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象  
4 Map类型。Map承载的数据必须是AIDL支持的类型，或者是其它声明的AIDL对象  
- 写服务端服务。AIDLService.java
```
public class AIDLService extends Service{
    private static final String TAG = "AIDLService";
    public AIDLService() {
    }
    
    //这里定义的aidl.Stub就是IBinder接口。这里模拟服务端处理耗时请求，之后通过回调onSuccess告诉客户端结果
    private IMyAidlInterface.Stub stub = new IMyAidlInterface.Stub() {
        @Override
        public void operation(int param1, int param2, IMyListener listener) {
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            try {
                listener.onSuccess(param1*param2);
            } catch (RemoteException e) {
                e.printStackTrace();
            }

        }


    };
    
    @Override
    public IBinder onBind(Intent intent) {
        Toast.makeText(this,"服务绑定成功",Toast.LENGTH_LONG).show();
        return stub;
    }
}
```
这样服务端就写好了。
-  客户端复制服务端的aidl文件，build后只需要调用服务端方法就可以
```
    private IMyAidlInterface iMyAidlInterface;
    private IMyListener mCallback = new IMyListener.Stub() {
        @Override
        public void onSuccess(int result) throws RemoteException {
            resultTv.setText("" + result);
        }
    };

    private void initView() {
        findViewById(R.id.botton).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {

                String params1 = params1Tv.getText().toString();
                String params2 = params2Tv.getText().toString();
                try {
                    if (iMyAidlInterface == null) {
                        Toast.makeText(MainActivity.this, "没注册", Toast.LENGTH_LONG).show();
                    } else {
                        iMyAidlInterface.operation(Integer.parseInt(params1), Integer.parseInt(params2), mCallback);
                    }

                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }
        });
        findViewById(R.id.bind).setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View v) {
                bindService();
            }
        });
    }

    //只能通过包名和aidl服务全名来绑定
    private void bindService() {
        Intent intent = new Intent();
        intent.setClassName("demo.beyond.com.blog", "demo.beyond.com.blog.service.AIDLService");
        bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE);
    }

    private ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, final IBinder service) {
            iMyAidlInterface = IMyAidlInterface.Stub.asInterface(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {
            iMyAidlInterface = null;
        }
    };
```
这里客户端只能通过bind方式绑定远程服务，onServiceConnected回调后端IBinder就是远程端AIDLService，客户端就可以调用服务端AIDLService实现端方法，服务端可以通过传进来的回调对象，调用回调方法onSuccess来通知客户端处理结果。  
Demo链接：  
[服务端 IMyAidlInterface.aidl](https://github.com/beyond667/study/blob/master/app/src/main/aidl/demo/beyond/com/blog/IMyAidlInterface.aidl)  
[AIDLService.java](https://github.com/beyond667/study/blob/master/app/src/main/java/demo/beyond/com/blog/service/AIDLService.java)  
[客户端 ClientActivity.java](https://github.com/beyond667/study/blob/master/app/src/main/java/demo/beyond/com/blog/service/ClientActivity.java) 

### 使用场景
一般普通的异步任务，比如网络请求，数据库或者文件相关操作，我们都会使用线程池的方式来做，因为这样使用的系统开销小，运行效率高，而且随着业务逻辑的复杂度增加，扩展性也更强。然而，对于一些特殊场景，比如进程保活，使用第三方SDK服务比如地图，IM等，就需要使用Service来实现，因为这些服务一般与App主进程隔离开，需要运行在新进程中以防止App主进程发生异常崩溃时，牵连第三方服务也挂掉。进程保活后面会专门研究。

