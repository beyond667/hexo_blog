---
title: Activity基础
date: 2019-05-24 22:00:37
tags: Android
---

### 前言
Activity作为四大组件之一，也可以说是四大组件中最重要的一个组件，下面我们从以下几点来理解activity。
- 显式启动和隐式启动
- 任务栈和Affinity
- 生命周期
- 启动模式

#### 显式启动和隐式启动
安卓有两种方式启动Activity，一种是显示启动，这个大家经常用到，不再细述。另外一种是隐式启动，隐式启动常用于不同应用之间的跳转（例如打开支付宝微信的支付页面），也可用于H5与native之间的交互。隐式启动就是action，category和data的匹配，我们先来说下匹配的规则，然后通过具体的例子去验证。

##### action的匹配规则
- action在Intent-filter可以设置多条
- intent中必须指定action否则匹配失败且intent中action最多只有一条
- intent中的action和intent-filter中的action必须完全一样时（包括大小写）才算匹配成功
- intent中的action只要与intent-filter其中的一条匹配成功即可

```
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>

    <intent-filter>
        <action android:name="com.jrmf360.action.ENTER"/>
        <action android:name="com.jrmf360.action.ENTER2"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</activity>
```
下面的两个intent都可以匹配上面MainActivity的action规则
```
Intent intent = new Intent("com.jrmf360.action.ENTER");
startActivity(intent);
Intent intent2 = new Intent("com.jrmf360.action.ENTER2");
startActivity(intent2);
```

##### category的匹配规则
- category在intent-filter和intent中都可以有多条
- intent中所有的category都必须在intent-filter中找到一样的（包括大小写）才算匹配成功
- 通过intent启动Activity的时候如果没有添加category会自动添加android.intent.category.DEFAULT，如果intent-filter中没有添加android.intent.category.DEFAULT则会匹配失败

##### data的匹配规则
data语法
```
<data 
  android:host="string"
  android:mimeType="string"
  android:path="string"
  android:pathPattern="string"
  android:pathPrefix="string"
  android:port="string"
  android:scheme="string"/>
```
举例：
```
scheme://host:port/path|pathPrefix|pathPattern
jrmf://jrmf360.com:8888/first

scheme：主机的协议部分，如jrmf
host：主机部分，如jrmf360.com
port： 端口号，如8888
path：路径，如first
pathPrefix：指定了部分路径，它会跟Intent对象中的路径初始部分匹配，如first
pathPattern：指定的路径可以进行正则匹配，如first
mimeType：处理的数据类型，如image/*
```
- intent-filter中可以设置多个data
- intent中只能设置一个data
- intent-filter中指定了data，intent中就要指定其中的一个data
- setType会覆盖setData，setData会覆盖setType，因此需要使用setDataAndType方法来设置data和mimeType

新建两个项目A和B，在A中添加几个button，点击button通过隐式启动打开B中的Activity
先来看看B中隐式启动的配置
```
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN"/>
        <category android:name="android.intent.category.LAUNCHER"/>
    </intent-filter>

    <intent-filter>
        <action android:name="com.jrmf360.action.ENTER"/>
        <category android:name="android.intent.category.DEFAULT"/>
        <data
            android:host="jrmf360.com"
            android:port="8888"
            android:scheme="jrmf"/>
    </intent-filter>
</activity>

<activity android:name=".FirstActivity">
</activity>

<activity android:name=".SecondActivity">
</activity>
```
B的MainActivity根据A项目传过来的参数跳转到指定的页面
```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        parseData();
    }

    private void parseData() {
        Uri data = getIntent().getData();
        if (data != null){
            String scheme = data.getScheme();
            String host = data.getHost();
            int port = data.getPort();
            String path = data.getPath();
            String query = data.getQuery();
            String message = data.getQueryParameter("message");
            Log.e(getClass().getSimpleName(),"scheme:"+scheme);
            Log.e(getClass().getSimpleName(),"host:"+host);
            Log.e(getClass().getSimpleName(),"port:"+port);
            Log.e(getClass().getSimpleName(),"path:"+path);
            Log.e(getClass().getSimpleName(),"query:"+query);

            if ("/first".equals(path)){
                FirstActivity.intent(this,message);
                finish();
            }else if ("/second".equals(path)){
                SecondActivity.intent(this,message);
                finish();
            }
        }
    }
}
```
A项目按钮的点击实现
```
@Override
public void onClick(View v) {
    int id = v.getId();
    if (id == R.id.btn_firstActivity){
        Intent intent = new Intent();
        intent.setAction("com.jrmf360.action.ENTER");
        intent.setData(Uri.parse("jrmf://jrmf360.com:8888/first?message=Hello FirstActivity"));
        startActivity(intent);

    }else if (id == R.id.btn_secondActivity){
        Intent intent = new Intent();
        intent.setAction("com.jrmf360.action.ENTER");
        intent.setData(Uri.parse("jrmf://jrmf360.com:8888/second?message=Hello SecondActivity"));
        startActivity(intent);
    }else if (id == R.id.btn_mainActivity){
        Intent intent = new Intent("com.jrmf360.action.ENTER");
        intent.setData(Uri.parse("jrmf://jrmf360.com:8888"));
        startActivity(intent);
    }
}
```
##### 关于android.intent.action.MAIN和android.intent.category.LAUNCHER
- android.intent.action.MAIN：程序最先启动的Activity可以给多个Activity设置
- android.intent.category.LAUNCHER：应用程序是否显示在桌面，可以给多个Activity配置
- android.intent.action.MAIN和android.intent.category.LAUNCHER同时设置会在launcher显示一个应用图标，单独设置android.intent.category.LAUNCHER不会出现图标，且一个应用程序最少要有一对。也可以设置多对，这样会在系统桌面出现过个应用程序图标，每个图标代表每个入口。


### 生命周期
7个方法：onCreate,onStart,onRestart,onResume,onPause,onStop,onDestory。
以A，B（透明），C（不透明）Activity为例：
A可见时:onCreate,onStart,onResume  
跳转到B:onPause
跳转到C:onPause,onStop
从C返回:onRestart,onStart,onResume
按Home到桌面:onPause,onStop
从桌面又到A:onRestart,onStart,onResume
A后退:onPause,onStop,onDestory