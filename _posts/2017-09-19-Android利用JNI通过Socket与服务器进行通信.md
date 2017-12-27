不多说，先上代码。记得点击star哦，代码地址是：SocketDemo
上一篇文章写了如何通过Java层实现Socket和服务器的Socket进行通信，这一篇继续深究，写个如何通过native层实现socket和服务器进行通信。服务器端代码和前一篇博客 代码一致，主要看下Android端的代码。首先看下Main2Activity的代码：
Main2Activity.java
``` java
package com.zqc.socketdemo;
import android.app.Activity;
import android.content.Context;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.view.View;
import android.widget.Button;
import android.widget.TextView;
public class Main2Activity extends Activity implements View.OnClickListener {  
    private TextView tv;  
    private Button bt;  
    Handler handler;  
    private Context mContext;  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
        setContentView(R.layout.activity_main2);  
        tv = (TextView) findViewById(R.id.textView);  
        bt = (Button) findViewById(R.id.button);  
        bt.setOnClickListener(this);  
        mContext = this;  
        handler = new Handler(Looper.getMainLooper());  
    }  
    Override  
    public void onClick(View v) {  
        switch (v.getId()) {  
            case R.id.button:  
                new Thread(){//网络连接需要在线程中实现  
                    @Override  
                    public void run() {  
                        SocketJNI.connect(mContext, "10.18.73.62", 6868);//和服务器进行socket通信  
                    }  
                }.start();  
                break;  
        }  
    }  
}
```
对应的xml布局是
activity_main2.xml
``` xml
<?xml version="1.0" encoding="utf-8"?>  
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"  
    xmlns:tools="http://schemas.android.com/tools"  
    android:id="@+id/activity_main2"  
    android:layout_width="match_parent"  
    android:layout_height="match_parent"  
    tools:context="com.zqc.socketdemo.Main2Activity">  
  
    <TextView  
        android:text="TextView"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_below="@+id/button"  
        android:layout_alignStart="@+id/button"  
        android:layout_marginTop="69dp"  
        android:id="@+id/textView" />  
  
    <Button  
        android:text="Button"  
        android:layout_width="wrap_content"  
        android:layout_height="wrap_content"  
        android:layout_marginTop="11dp"  
        android:id="@+id/button"  
        android:layout_marginStart="27dp"  
        android:layout_alignParentTop="true"  
        android:layout_alignParentStart="true" />  
</RelativeLayout>  
``` 
其中SocketJNI.java的代码很简单，代码如下：
``` java
package com.zqc.socketdemo;  
  
import android.content.Context;  
  
/** 
 * Created by zhangqianchu on 2017/8/2. 
 */  
  
public class SocketJNI {  
    static {  
        System.loadLibrary("mysocket");  
    }  
    public native static void connect(Context mContext, String ip, int port);//这里定义了一个jni方法，在native层实现socket连接  
}  
``` 
在SocketJNI.java类中定义了一个jni方法，即提供一个Java层的调用，具体代码在native层实现。下面看看native层的代码
``` cpp
#include <jni.h>  
#include <string.h>  
#include "socketutil.h"  
  void showToast(JNIEnv *env, jobject context, jstring str) {  
    jclass tclss = (*env)->FindClass(env,"android/widget/Toast");    
     jmethodID mid = (*env)->GetStaticMethodID(env,tclss,"makeText","(Landroid/content/Context;Ljava/lang/CharSequence;I)Landroid/widget/Toast;");    
     jobject job = (*env)->CallStaticObjectMethod(env,tclss,mid,context,str);    
     jmethodID showId = (*env)->GetMethodID(env,tclss,"show","()V");    
     (*env)->CallVoidMethod(env,job,showId,context,str);    
  }  
  
 JNIEXPORT void JNICALL Java_com_zqc_socketdemo_SocketJNI_connect    
   (JNIEnv *env, jclass jcz, jobject context, jstring addr, jint port) {     
    jstring msg = (*env)->NewStringUTF(env,"start");  
    //showToast(env,context,msg);//show的时候会出现暂时卡顿，原因不明  
    LOGI("connect socket.");    
    const char* response =  connectRemote("10.18.73.62", 6868); //response就是服务端传给客户端的内容  
    release();    
    LOGI("connect socket end.");    
    //msg  = (*env)->NewStringUTF(env,response);  
    //showToast(env,context,msg);/  
  }
``` 
这里将socket的具体实现放到socketutil.h里面进行实现，避免代码耦合。
socketutil.h
``` cpp
#ifndef SOCKET_UTIL  
#define SOCKET_UTIL  
#include <jni.h>  
#include <android/log.h>    

#define  LOG_TAG    "mysocket"    
#define  LOGI(...)  __android_log_print(ANDROID_LOG_INFO,LOG_TAG,__VA_ARGS__)    

void release();  
const char* connectRemote(const char*,const int);  
#endif  
socketutil.c
[cpp] view plain copy
#include <jni.h>  
#include <stdio.h>  
#include <string.h>  
#include <android/log.h>  
#include "socketutil.h"  
#include <sys/socket.h>    
#include <netinet/in.h>    
#include <stdlib.h>    
#include <android/log.h>    
  
#define MAXSIZE 4096    
struct sockaddr_in sock_add;    
struct sockaddr_in server_addr;    
int sockfd = 0;    
    
void release(){    
   close(sockfd);    
}    
    
const char* connectRemote(const char* addr,const int port) {   
     sockfd = socket(AF_INET, SOCK_STREAM, 0);    //ipv4,TCP数据连接    
     LOGI("step1");    
     if (sockfd < 0) {    
        return "socket error";    
     }    
    //服务器地址   
     bzero(&server_addr,sizeof(server_addr));    
     server_addr.sin_family = AF_INET;    
     server_addr.sin_port = htons(port);    
     LOGI("step2");    
     if( inet_pton(AF_INET, addr, &server_addr.sin_addr) < 0){    //设置ip地址    
       LOGI("address error");  
       return "address error";    
     }    
     socklen_t server_addr_length = sizeof(server_addr);    
     int connfd = connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr)); //连接到服务器    
     LOGI("step3");    
     if (connfd < 0) {    
        return "connect error";    
     }    
     //发送数据    
     char *msg = "i send a data to server. \n";    
     int sdf = send(sockfd, msg , strlen(msg), 0); //发送一行数据到服务器    
     LOGI("step4.");    
     char buff[4096];    
     int n = recv(sockfd, buff, MAXSIZE ,0);  //这地方应该需要循环获取数据，目前服务器只响应了一条很短的字符串。    
     buff[n] = 0;    
     LOGI("step5.");    
     return buff;    
}
``` 
这里的socket连接使用的是最基本的Socket连接方法，首先设置协议（IPV4协议），然后设置ip地址和port端口号，之后便尝试连接服务器，调用的连接方法是sys/socket.h库中的函数，连接成功后会返回一个int值，若该值小于0则表示连接出错。
上述代码编写好之后，再编写个Android.mk，然后利用ndk-build命令将c代码编译为so库，放入到工程的libs目录下。其中Android.mk代码如下：
Android.mk
``` 
LOCAL_PATH := $(call my-dir)  
  
include $(CLEAR_VARS)  
LOCAL_CPP_EXTENSION :=cpp #针对C++的支持，标记c++文件的扩展名名称  
LOCAL_MODULE := mysocket  
LOCAL_SRC_FILES := socketutil.c \  
                   socketjni.c  
LOCAL_LDLIBS := -llog  
include $(BUILD_SHARED_LIBRARY)  
``` 
为了少编译些版本，也可以加入一个Application.mk，代码如下：
Application.mk
``` 
#APP_MODULES := mysocket //so库的名称，在Android.mk里面设置了  
#APP_STL:=stlport_static  
APP_ABI := armeabi #编译的版本有哪些  
#APP_OPIM :=debug  
``` 
 利用ndk-build命令进行编译，ndk-build命令会直接将so库放入到libs目录下，然后运行工程，点击里面的Button按钮，即可查看连接结果。注意，手机要和服务端在同一个局域网里面，而且在运行的时候要修改下代码里面的ip地址。
运行后输出的Log如下：

上面的step1到step5表示运行到了哪一步，方便查看具体是在哪一步出错了。
服务端输出如下：

