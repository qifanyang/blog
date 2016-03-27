---
title: jni数据类型
date: 2016-03-23 17:41:55
tags: java
---
## 本地方法
java一些平台相关或者和虚拟机相关的方法需要使用本地方法实现,例如

	java.lang.Thread.java
	private static native void registerNatives();

java.lang.Thread.java对应一个java.lang.Thread.c,实现如下

	JNIEXPORT void JNICALL
	Java_java_lang_Thread_registerNatives(JNIEnv *env, jclass cls)
	{
    	(*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));
	}

## JNIEnv *env
对于c语言来讲,JNIEnv是个结构体,对于c++来讲,JNIEnv是个类,在Thread.c中代码也针对编译器做了兼容,不论是c编译器还是c++编译器都可以编译Thread.c;

	#ifdef __cplusplus
		typedef JNIEnv_ JNIEnv;
	#else
		typedef const struct JNINativeInterface_ *JNIEnv;
	#endif
如果采用c编译器编译,JNIEnv则是一个JNINativeInterface_结构体指针,其声明如下

	struct JNINativeInterface_ {
    void *reserved0;
    void *reserved1;
    void *reserved2;

    void *reserved3;
    jint (JNICALL *GetVersion)(JNIEnv *env);

    jclass (JNICALL *DefineClass)
      (JNIEnv *env, const char *name, jobject loader, const jbyte *buf,
       jsize len);
    jclass (JNICALL *FindClass)
      (JNIEnv *env, const char *name);
	...

JNINativeInterface_结构体中定义了很多函数指针,JNIEnv本身就是一个指针,所以JNIEnv *env是一个指向JNIEnv的指针(指针的指针);

(*env)->RegisterNatives(env, cls, methods, ARRAY_LENGTH(methods));

(*env)相当于获取到JNIEnv, 而->为指针解引用,RegisterNatives是一个函数指针变量,在结构体中包含一个函数指针可以模拟面向对象的方法调用


## jclass
java中的Class对象在虚拟机中引用的指针,在java中数据类型在虚拟机中都有一个对应的数据类型,struct _jobject;是可以指向任何java数据类型的结构
typedef好处:1.可移植性 2.程序参数化

	struct _jobject;

	typedef struct _jobject *jobject;
	typedef jobject jclass;
	typedef jobject jthrowable;
	typedef jobject jstring;
	typedef jobject jarray;
	typedef jarray jbooleanArray;
	typedef jarray jbyteArray;
	typedef jarray jcharArray;
	typedef jarray jshortArray;
	typedef jarray jintArray;

	typedef unsigned char   jboolean;
	typedef unsigned short  jchar;
	typedef short           jshort;
	typedef float           jfloat;
	typedef double          jdouble;
	//jint平台依赖不在jni.h中,在jni_md.h中声明


## JNINativeMethod
java本地方法c语言表示,结构体如下:

	typedef struct {
    	char *name;
		char *signature;
    	void *fnPtr;
	} JNINativeMethod;

