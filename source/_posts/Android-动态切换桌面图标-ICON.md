---
title: Android 动态切换桌面图标 ICON
date: 2023-09-04 10:45:59
tags:
---


谷歌认为，图标变更功能应该使用独立的LAUNCHER Activity实现，而不应借助activity-alias。
最建议方案如下：

### 1. 新建 NewMainActivity()
```kotlin
class NewMainActivity : MainActivity()
```

### 2. 配置 AndroidManifest.xml
```xml
<activity
            android:name=".MainActivity"
            android:enabled="true"
            android:exported="true"
            android:icon="@mipmap/ic_launcher"
            android:label="@string/app_name"
            android:targetActivity=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <activity
            android:name=".NewMainActivity"
            android:enabled="false"
            android:exported="true"
            android:icon="@mipmap/ic_launcher_show"
            android:label="@string/app_name"
            android:targetActivity=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
```

### 3. MainActivity() 中切换代码
```java
open class MainActivity : AppCompatActivity() {
    private var position = 0
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        findViewById<View>(R.id.btn_default).setOnClickListener {
            position = 0
//            setDefaultAlias()
            Toast.makeText(this@MainActivity, "退到后台后切换", Toast.LENGTH_SHORT).show()
        }
        findViewById<View>(R.id.btn_alias1).setOnClickListener {
            position = 1
//            setAlias1()
            Toast.makeText(this@MainActivity, "退到后台后切换", Toast.LENGTH_SHORT).show()
        }

        ProcessLifecycleOwner.get().lifecycle.addObserver(object : LifecycleEventObserver {
            override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
                when (event) {
                    Lifecycle.Event.ON_RESUME -> {
                    }

                    Lifecycle.Event.ON_START -> {
                        Log.i("LifecycleObserver", "应用回到前台")
                    }

                    Lifecycle.Event.ON_STOP -> {
                        Log.i("LifecycleObserver", "应用退到后台")
                        //根据具体业务需求设置切换条件,我公司采用接口控制icon切换
                        if (position == 0) {
                            setDefaultAlias()
                        } else {
                            setAlias1()
                        }
                    }

                    else -> {}
                }
            }
        })
    }

    /**
     * 设置默认的别名为启动入口
     * 在设置图标不可见的方法中，我们传递的是 PackageManager.DONT_KILL_APP ，在设置图标可见的方法中我们传递的是 0 。传递0表示会杀死包含该组件的app，桌面图标立即会更改掉。
     */
    fun setDefaultAlias() {
        val packageManager = packageManager
        val name2 = ComponentName(this, NewMainActivity::class.java)
        packageManager.setComponentEnabledSetting(
            name2,
            PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP
        )
        val name1 = ComponentName(this, MainActivity::class.java)
        packageManager.setComponentEnabledSetting(
            name1,
            PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
            0
        )
    }

    /**
     * 设置别名1为启动入口
     * 在设置图标不可见的方法中，我们传递的是 PackageManager.DONT_KILL_APP ，在设置图标可见的方法中我们传递的是 0 。传递0表示会杀死包含该组件的app，桌面图标立即会更改掉。
     */
    fun setAlias1() {
        val packageManager = packageManager
        val name1 = ComponentName(this, MainActivity::class.java)
        packageManager.setComponentEnabledSetting(
            name1,
            PackageManager.COMPONENT_ENABLED_STATE_DISABLED,
            PackageManager.DONT_KILL_APP
        )
        val name2 = ComponentName(this, NewMainActivity::class.java)
        packageManager.setComponentEnabledSetting(
            name2,
            PackageManager.COMPONENT_ENABLED_STATE_ENABLED,
            0
        )
    }
}
```
### 4. 参考资料
- https://www.jianshu.com/p/c380f54e71bf

