---
title: openJDK source code directory
date: 2016-03-04 10:01:56
tags: openJDK
---
### 目录结构

![](/img/OpenJDK目录.png)




JNI变量定义

typedef signed char jbyte; //所以java没有无符号类型
typedef int jint;

JNIEXPORT //很多方法以这个开头
平台依赖定义如下:
solaris
#if (defined(__GNUC__) && ((__GNUC__ > 4) || (__GNUC__ == 4) && (__GNUC_MINOR__ > 2))) || __has_attribute(visibility)
  #define JNIEXPORT     __attribute__((visibility("default")))
  #define JNIIMPORT     __attribute__((visibility("default")))
#else
  #define JNIEXPORT
  #define JNIIMPORT
#endif
#define JNICALL

windows
#define JNIEXPORT __declspec(dllexport)
#define JNIIMPORT __declspec(dllimport)
#define JNICALL __stdcall

一个jni方法定义:
JNIEXPORT void JNICALL
Java_java_lang_Object_registerNatives(JNIEnv *env, jclass cls)
{
    (*env)->RegisterNatives(env, cls,
                            methods, sizeof(methods)/sizeof(methods[0]));
}

struct JNIInvokeInterface_; //JNI调用接口
struct JNIInvokeInterface_ {
    void *reserved0;
    void *reserved1;
    void *reserved2;

    jint (JNICALL *DestroyJavaVM)(JavaVM *vm);

    jint (JNICALL *AttachCurrentThread)(JavaVM *vm, void **penv, void *args);

    jint (JNICALL *DetachCurrentThread)(JavaVM *vm);

    jint (JNICALL *GetEnv)(JavaVM *vm, void **penv, jint version);

    jint (JNICALL *AttachCurrentThreadAsDaemon)(JavaVM *vm, void **penv, void *args);
};


struct JNINativeInterface_ {//采用结构体加函数指针,实现成员方法的调用效果.方法实现在哪里呢?
    void *reserved0;
    void *reserved1;
    void *reserved2;

    void *reserved3;
    jint (JNICALL *GetVersion)(JNIEnv *env);

    jclass (JNICALL *DefineClass)
      (JNIEnv *env, const char *name, jobject loader, const jbyte *buf,
       jsize len);

	...//
}