---
title: android夸进程通信-AIDL
date: 2023-08-26 21:55:43
tags: android
categories: android
top_img: https://s2.loli.net/2023/08/26/GjXxt3wIDgW8Js2.png
cover: https://s2.loli.net/2023/08/26/GjXxt3wIDgW8Js2.png

---



## 1.在服务端创建ALDL文件

<img src="https://s2.loli.net/2023/08/26/GjXxt3wIDgW8Js2.png" alt="image-20230826220210672" style="zoom:50%;" />



会生成AIDL文件及接口文件

<img src="https://s2.loli.net/2023/08/26/tcmE43rN5TfUaF9.png" alt="image-20230826220434517" style="zoom:50%;" />

```kotlin
// IMyAidlInterface.aidl
package com.example.startservicefromanotherapp;

// Declare any non-default types here with import statements

interface IMyAidlInterface {

    void basicTypes(int anInt, long aLong, boolean aBoolean, float aFloat,
            double aDouble, String aString);

     void SetData(String str); //这是我自己写的 以后会用来接收消息
}
```

## 2.创建Server组件

<img src="https://s2.loli.net/2023/08/26/34U9uDrqpVSQOeM.png" alt="image-20230826220746473" style="zoom:50%;" />

```kotlin
package com.example.startservicefromanotherapp

import android.app.Service
import android.content.Intent
import android.content.ServiceConnection
import android.nfc.Tag
import android.os.Binder
import android.os.IBinder
import android.os.IInterface
import android.os.Parcel
import android.util.Log
import java.io.FileDescriptor
import kotlin.concurrent.thread
import kotlinx.coroutines.*
class AppService : Service() {


    private val TAG="HeiBao"
    var str1:String="我是一只来自北方的狼"
    /**
     * 返回一个IBinder对像,服务器的代理类
     * 返回对象,是Activity和当前Service沟通/交互的唯一渠道
     */
    override fun onBind(intent: Intent): IBinder? {
        Log.d(TAG, "onBind: 调用OnBind方法")
        return MyAidllnterface() //这里返回AIDL接口对象
    }
//    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
//        Log.d(TAG, "Service start111")
//        return START_STICKY
//    }
    override fun onCreate() {
        super.onCreate()
    GlobalScope.launch {

        while(true) {
            delay(1000)

            Log.d(TAG,"输出结果是:${str1}")
        }

    }
        Log.d(TAG,"Service start")
    }

    override fun onDestroy() {
        super.onDestroy()

        Log.d(TAG,"Service Destroy")
    }

    override fun unbindService(conn: ServiceConnection) {
        super.unbindService(conn)

        Log.d(TAG,"unbindService")
    }
    /**
     * 自定义MyAidllnterface 继承 Binder
     */

    inner class MyAidllnterface: IMyAidlInterface.Stub() {
        override fun basicTypes(
            anInt: Int,
            aLong: Long,
            aBoolean: Boolean,
            aFloat: Float,
            aDouble: Double,
            aString: String?,
        ) {
            TODO("Not yet implemented")
        }

        override fun SetData(str: String?) {
            if (str != null) {
                str1=str
            }
        }

    }
}
```

## 3.在客服端创建AIDL 

   这时不要创建AIDL文件了,只是创建他的文件夹,然后在创建一个和服务端的package一样的名子 然后把服务端的IMyAidlInterface.aidl 复制到package下面 ,点到android studio中Rebulid Project

<img src="https://s2.loli.net/2023/08/26/pMRltvnbW79dEsQ.png" alt="image-20230826221738240" style="zoom:50%;" />

<img src="https://s2.loli.net/2023/08/26/3UtkiKHA7c8Is6Y.png" alt="image-20230826221654478" style="zoom:50%;" />

以下的客户端代码

```kotlin
package com.example.anotherapp

import android.annotation.SuppressLint
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.content.ServiceConnection
import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.os.IBinder
import android.util.Log
import android.view.View
import android.view.View.OnClickListener

import android.widget.Button
import android.widget.Toast
import com.example.anotherapp.databinding.ActivityMainBinding
import com.example.anotherapp.databinding.ActivityMainBinding.inflate
import com.example.startservicefromanotherapp.IMyAidlInterface

class MainActivity : AppCompatActivity(),OnClickListener {


    private val TAG="HeiBao"

    private lateinit var viewBinding: ActivityMainBinding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        viewBinding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(viewBinding.root)

        viewBinding.button1.setOnClickListener(this)
        viewBinding.button2.setOnClickListener(this)
    }
    private val conn = object : ServiceConnection {

        /**
         * 当成功绑定服务时调用的方法
         * 该方法可能以获取AppService 中的OnBinder 的返回对象
         */
        @SuppressLint("SuspiciousIndentation")
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            // 绑定成功回调

         val mYidInterface =   IMyAidlInterface.Stub.asInterface(service)

            Log.e(TAG, "onServiceConnected服务绑定成功: ${name.toString()} ${mYidInterface.SetData("666666")} ", )
        }

        override fun onServiceDisconnected(name: ComponentName) {
            // 绑定断开回调
            Log.e(TAG, "onServiceDisconnected: ", )
        }

    }

        override fun onClick(v: View?) {
            when (v?.id) {
                R.id.button2 -> {
                    //Toast.makeText(this, "1+1=2", Toast.LENGTH_SHORT).show()
                    unbindService(conn);
                }
                R.id.button1 -> {

//                    val intent = Intent()
//
//                    intent.setAction("com.example.startservicefromanotherapp.ACTION_APP_SERVICE")
//                    intent.setPackage("com.example.startservicefromanotherapp")
//                    startForegroundService(intent)
//
                    val intent = Intent()
                    intent.setComponent(ComponentName("com.example.startservicefromanotherapp", "com.example.startservicefromanotherapp.AppService"))
                   // startService(intent)

                    bindService(intent, conn, Context.BIND_AUTO_CREATE)

                }


            }
        }




}
```

具体查看 StartServiceFromAnotherApp项目源码



https://github.com/ljm2007/douyu/blob/master/StartServiceFromAnotherApp.zip