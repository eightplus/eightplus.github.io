---
title:      "在Android源码中使用java编写自己的应用程序"
date:       2018-08-09 21:49:29
author:     "lixiang"
categories: 工具
tags:
    - Android
---

## HelloWorld

- 程序目录结构如下：
```
~/Android/packages/experimental/HelloWorld
----AndroidMainifest.xml
----Android.mk
----src
    ----com/eightplus/mydev
        ----HelloWorld.java
----res
    ----layout
        ----main.xml
    ----values
        ----strings.xml
    ----drawable
        ----icon.png
```

- HelloWorld.java

``` java
package com.eightplus.hello

import android.app.Activity;
import android.os.Bundle;
import android.util.log;

public class HelloWorld extends Activity {
    private final static String LOG_TAG = "HelloWorld";

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        Log.i(LOG_TAG, "HelloWorld, Welcome!!!");
    }
}
```

- main.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
	android:orientation="vertical"
	android:layout_width="fill_parent"
	android:layout_height="fill_parent"
	android:gravity="center">
		<TextView
			android:layout_width="wrap_content"
			android:layout_height="wrap_content"
			android:gravity="center"
			android:text="@string/hello_world" >
		</TextView>
</LinearLayout>
```

- strings.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
	<string name="app_name">HellWorld</string>
	<string name="hello_world">Hello World</string>
</resources>
```

- icon.png是应用程序的图标

- AndroidMainifest.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<mainifest xmlns:android="http://schemas.android.com/apk/res/android"
	package="com.eightplus.hello"
	android:versionCode="1"
	android:versionName="1.0">
	<application android:icon="@drawable/icon"
		android:label="@string/app_name">
		<activity android:name=".HelloWorld"
			android:label="@string/app_name">
			<intent-filter>
				<action android:name="android.intent.action.MAIN" />
				<category android:name="android.intent.category.LAUNCHER" />
			</intent-filter>
		</activity>
	</application>
</mainifest>
```

- Android.mk
```
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_PACKAGE_NAME := HelloWorld
include $(BUILD_PACKAGE)
```

- 使用mmm命令单独编译HelloWorkd程序
* ~/Android$ mmm ./packages/experimental/HelloWorld/
编译完成之后，就可以在out/target/product/generic/system/app目录下看到编译结果的输出文件HelloWorld.apk了。

- 重新打包Android系统文件system.img
* ~/Android$ make snod

- 运行模拟器
* ~/Android$ emulator

---

## Activity组件应用实例

该实例由三个Acitity组件组成：MainActivity、SubActivityInProcess和SubActivityInNewProcess。MainActivity是根Acitity，SubActivityInProcess和SubActivityInNewProcess是子Acitity。MainActivity和SubActivityInProcess运行在同一个进程中，而SubActivityInNewProcess运行在一个独立的进程中。

- 程序目录结构如下：
```
~/Android/packages/experimental/Activity
----AndroidMainifest.xml
----Android.mk
----src
    ----com/eightplus/activity
        ----MainActivity.java
        ----SubActivityInProcess.java
        ----SubActivityInNewProcess.java
----res
    ----layout
        ----main.xml
        ----sub.xml
    ----values
        ----strings.xml
    ----drawable
        ----icon.png
```

- MainActivity.java

SubActivityInProcess和SubActivityInNewProcess的组件名字分别被配置为"com.eightplus.activity.subactivity.in.process"和"com.eightplus.activity.subactivity.in.new.process"，因此调用成员和函数startActivity启动它们时，只需要分别指定这两个名称即可，不需要知道它们是哪个类来实现的。
``` java
package com.eightplus.activity;  

import android.app.Activity;  
import android.content.Intent;  
import android.os.Bundle;  
import android.util.Log;  
import android.view.View;  
import android.view.View.OnClickListener;  
import android.widget.Button;  

public class MainActivity extends Activity  implements OnClickListener {  
    private final static String LOG_TAG = "com.eightplus.activity.MainActivity";  

    private Button startInProcessButton = null;
	private Button startInNewProcessButton = null;

    @Override  
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.main);  

        startInProcessButton = (Button)findViewById(R.id.button_start_in_process);
        startInNewProcessButton = (Button)findViewById(R.id.button_start_in_new_process);
        startInProcessButton.setOnClickListener(this);
        startInNewProcessButton.setOnClickListener(this);

        Log.i(LOG_TAG, "Main Activity Created.");  
    }  

    @Override  
    public void onClick(View v) {  
        if(v.equals(startInProcessButton)) {  
            Intent intent = new Intent("com.eightplus.activity.subactivity.in.process");  
            startActivity(intent);  
        }
        else if(v.equals(startInNewProcessButton)) {  
            Intent intent = new Intent("com.eightplus.activity.subactivity.in.new.process");  
            startActivity(intent);  
        }
    }  
}
```
- SubActivityInProcess.java

``` java
package com.eightplus.activity;  

import android.app.Activity;  
import android.os.Bundle;  
import android.util.Log;  
import android.view.View;  
import android.view.View.OnClickListener;  
import android.widget.Button;  

public class SubActivityInProcess extends Activity implements OnClickListener {  
    private final static String LOG_TAG = "com.eightplus.activity.SubActivityInProcess";  

    private Button finishButton = null;  

    @Override  
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.sub);  

        finishButton = (Button)findViewById(R.id.button_finish);  
        finishButton.setOnClickListener(this);  

        Log.i(LOG_TAG, "Sub Activity In Process Created.");  
    }  

    @Override  
    public void onClick(View v) {  
        if(v.equals(finishButton)) {  
            finish();  
        }  
    }  
}
```
- SubActivityInNewProcess.java

``` java
package com.eightplus.activity;  

import android.app.Activity;  
import android.os.Bundle;  
import android.util.Log;  
import android.view.View;  
import android.view.View.OnClickListener;  
import android.widget.Button;  

public class SubActivityInNewProcess extends Activity implements OnClickListener {  
    private final static String LOG_TAG = "com.eightplus.activity.SubActivityInNewProcess";  

    private Button finishButton = null;  

    @Override  
    public void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.sub);  

        finishButton = (Button)findViewById(R.id.button_finish);  
        finishButton.setOnClickListener(this);  

        Log.i(LOG_TAG, "Sub Activity In New Process Created.");  
    }  

    @Override  
    public void onClick(View v) {  
        if(v.equals(finishButton)) {  
            finish();  
        }  
    }  
}
```

- main.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:orientation="vertical"  
    android:layout_width="fill_parent"  
    android:layout_height="fill_parent"   
    android:gravity="center">  
        <Button   
            android:id="@+id/button_start_in_process"  
            android:layout_width="wrap_content"  
            android:layout_height="wrap_content"  
            android:gravity="center"  
            android:text="@string/start_in_process" >  
        </Button>
        <Button   
            android:id="@+id/button_start_in_new_process"  
            android:layout_width="wrap_content"  
            android:layout_height="wrap_content"  
            android:gravity="center"  
            android:text="@string/start_in_new_process" >  
        </Button>
</LinearLayout>
```

- sub.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:orientation="vertical"  
    android:layout_width="fill_parent"  
    android:layout_height="fill_parent"   
    android:gravity="center">  
        <Button   
            android:id="@+id/button_finish"  
            android:layout_width="wrap_content"  
            android:layout_height="wrap_content"  
            android:gravity="center"  
            android:text="@string/finish" >  
        </Button>
</LinearLayout>
```

- strings.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Activity</string>
    <string name="sub_activity">Sub Activity</string>
    <string name="start_in_process">Start sub-activity in process</string>
    <string name="start_in_new_process">Start sub-activity in new process</string>
    <string name="finish">Finish activity</string>
</resources>
```

- icon.png为应用程序的图标

- AndroidMainifest.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.eightplus.activity"
    android:versionCode="1"
    android:versionName="1.0">
    <application android:icon="@drawable/icon" android:label="@string/app_name">
        <activity android:name=".MainActivity"
              android:label="@string/app_name"
              android:process="com.eightplus.activity.mainprocess">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <activity android:label="@string/sub_activity"
                  android:name=".SubActivityInProcess"
                  android:process="com.eightplus.activity.mainprocess">
            <intent-filter>
                <action android:name="com.eightplus.activity.subactivity.in.process"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </activity>
        <activity android:label="@string/sub_activity"
                  android:name=".SubActivityInNewProcess"
                  android:process="com.eightplus.activity.newprocess">
            <intent-filter>
                <action android:name="com.eightplus.activity.subactivity.in.new.process"/>
                <category android:name="android.intent.category.DEFAULT"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

- Android.mk
```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_PACKAGE_NAME := Activity
include $(BUILD_PACKAGE)
```

- 编译和打包
* ~/Android$ mmm packages/experimental/Activity    
* ~/Android$ make snod

- 启动模拟器
* ~/Android$ emulator

---

## Service组件应用实例

该实例由一个Service组件CounterService和一个Activity组件Counter组成。CounterService组件实现了一个计数器服务，它是由Counter组件启动起来的，并且与Counter组件绑定在一起。CounterService组件在内部使用一个异步任务(AsyncTask)来实现一个计数器，每隔一秒就将内部的计数加1,并且实时地将这个计数显示在Counter组件的用户界面上。由于计数器需要不停地执行加数功能，所以这里将它放在一个后台程序中运行，避免Counter组件不能及时地响应用户界面事件。

- 程序目录结构如下：
```
~/Android/packages/experimental/Counter
----AndroidMainifest.xml
----Android.mk
----src
    ----com/eightplus/counter
        ----ICounterCallback.java
        ----ICounterService.java
        ----CounterService.java
        ----Counter.java
----res
    ----layout
        ----main.xml
        ----sub.xml
    ----values
        ----strings.xml
    ----drawable
        ----icon.png
```

- ICounterCallback.java

``` java
package com.eightplus.counter

publicd interface ICounterCallback {
    void count(int val);
}
```

- ICounterService.java

``` java
package com.eightplus.counter

publicd interface ICounterService {
    public void startCounter(int initVal, ICounterCallback callback);
    public void stopCounter();
}
```

- CounterService.java

``` java
package com.eightplus.counter

import android.app.Service;
import android.content.Intent;
import android.os.AsyncTask;
import android.os.Binder;
import android.os.IBinder;
import android.util.Log;

public class CounterService extends Service implements ICounterService {
    private final static String LOG_TAG = "com.eightplus.activity.CounterService";  

    private bool stop = false;
    private ICounterCallback counterCallback = null;

    private final IBinder binder = new CounterBinder();

    public class CounterBinder extends Binder {
        public CounterService getService() {
            return CounterService.this;
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return binder;
    }

    @Override
    public void onCreate() {
        super.onCreate();
        Log.i(LOG_TAG, "Counter Service Created.");
    }

    public void startCounter(int initVal, ICounterCallback callback) {
        counterCallback = callback;
        AsyncTask<Integer, Integer, Integer> task = new AsyncTask<Integer, Integer, Integer>() {
            @Override
            protected Integer doInBackground(Integer... vals) {
                Integer initCounter = vals[0];

                stop = false;
                while(!stop) {
                    publishProcess(initCounter);
                    try {
                        Thread.sleep(1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                    initCounter++;
                }
                return initCounter;
            }

            @Override
            protected void onProgressUpdate(Integer... values) {
                super.onProgressUpdate(values);
                int val = values[0];
                counterCallback.count(val);
            }

            @Override
            protected void onPostExecute(Integer val) {
                counterCallback.count(val);
            }
        };
        task.execute(initVal);
    }

    public void stopCounter() {
        stop = true;
    }
}
```

- Counter.java

``` java
package com.eightplus.counter

import android.app.Activity;
import android.content.ComponentName;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.content.ServiceConnection;
import android.os.Bundle;
import android.os.IBinder;
import android.util.Log;
import android.view.View;
import android.view.onClickListener;
import android.widget.Button;
import android.widget.TextView;

public class Counter extends Activity implements OnClickListener, ICounterCallback {
    private final static String LOG_TAG = "com.eightplus.activity.Counter";

    private Button startButton = null;
    private Button stopButton = null;
    private TextView counterText = null;

    private ICounterService counterService = null;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);

        startButton = (Button)findViewById(R.id.button_start);
        stopButton = (Button)findViewById(R.id.button_stop);
        counterText = (TextView)findViewById(R.id.textview_counter);

        startButton.setOnClickListener(this);
        stopButton.setOnClickListener(this);
        startButton.setEnabled(true);
        stopButton.setEnabled(false);

        Intent bindIntent = new Intent(Counter.this, CounterService.class);
        bindService(bindIntent, serviceConnection, Context.BIND_AUTO_CREATE);

        Log.i(LOG_TAG, "Counter Activity Created.");
    }

    @Override
    public void onDestroy() {
        super.onDestory();
        unbindService(serviceConnection);
    }

    @Override
    public void onClick(View v) {
            if(v.equals(startButton)) {  
                if (counterService != null) {
                    counterService.startCounter(0, this);
                    startButton.setEnabled(false);
                    stopButton.setEnabled(true);
                }
            } else if(v.equals(stopButton)) {  
                if (counterService != null) {
                    counterService.stopCounter();
                    startButton.setEnabled(true);
                    stopButton.setEnabled(false);
                }
            }
    }

    @Override
    public void count(int val) {
        String text = String.valueOf(val);
        counterText.setText(text);
    }

    private ServiceConnection serviceConnection = new ServiceConnection() {
        public void onServiceConnected(ComponentName className, IBinder service) {
        counterService = ((CounterService.CounterBinder)service).getService();
        Log.i(LOG_TAG, "Counter Service Connected");
    }

    public void onServiceDisconnected(ComponentName className) {
        counterService = null;
        Log.i(LOG_TAG, "Counter Service DisConnected");
    }
}
```

- main.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>  
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    android:orientation="vertical"  
    android:layout_width="fill_parent"
    android:layout_height="fill_parent"
    android:gravity="center">  
        <LinearLayout android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:layout_marginBottom="10px"
            android:orientation="horizontal"
            android:gravity="center">
		<TextView android:layout_width="wrap_content"
		    android:layout_height="wrap_content"
		    android:layout_marginBottom="4px"
		    android:gravity="center"
		    android:text="@string/counter">
		</TextView>
		<TextView android:id="@+id/textview_counter"
		    android:layout_width="wrap_content"
		    android:layout_height="wrap_content"
		    android:gravity="center"
		    android:text="@string/init_counter">
		</TextView>
        </LinearLayout>
        <LinearLayout android:layout_width="fill_parent"
            android:layout_height="wrap_content"
            android:orientation="horizontal"
            android:gravity="center">
		<Button android:id="@+id/button_start"
		    android:layout_width="wrap_content"
		    android:layout_height="wrap_content"
		    android:gravity="center"
		    android:text="@string/start">
		</Button>
		<Button android:id="@+id/button_stop"
		    android:layout_width="wrap_content"
		    android:layout_height="wrap_content"
		    android:gravity="center"
		    android:text="@string/stop">
		</Button>
        </LinearLayout>
</LinearLayout>
```

- strings.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <string name="app_name">Counter</string>
    <string name="counter">Counter:</string>
    <string name="init_counter">0</string>
    <string name="start">Start Counter</string>
    <string name="stop">Stop Counter</string>
</resources>
```

- icon.png为应用程序的图标

- AndroidMainifest.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.eightplus.counter"
    android:versionCode="1"
    android:versionName="1.0">
    <application android:icon="@drawable/icon" android:label="@string/app_name">
        <activity android:name=".Counter"
              android:label="@string/app_name"
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service android:name=".CounterService"
                  android:enabled="true">
        </service>
    </application>
</manifest>
```

- Android.mk
```
LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE_TAGS := optional
LOCAL_SRC_FILES := $(call all-subdir-java-files)
LOCAL_PACKAGE_NAME := Counter
include $(BUILD_PACKAGE)
```

- 编译和打包
* ~/Android$ mmm ./packages/experimental/Counter/    
* ~/Android$ make snod

- 启动模拟器
* ~/Android$ emulator
