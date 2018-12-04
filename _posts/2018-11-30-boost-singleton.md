---
layout: post
title:  "boost 单例模式源码解析"
categories: boost c++ 单例模式
---

* content
{:toc}

## 源码
```
// T must be: no-throw default constructible and no-throw destructible
template <typename T>
struct singleton_default
{
  private:
    struct object_creator
    {
      // This constructor does nothing more than ensure that instance()
      //  is called before main() begins, thus creating the static
      //  T object before multithreading race issues can come up.
      object_creator() { singleton_default<T>::instance(); }
      inline void do_nothing() const { }
    };
    static object_creator create_object;

    singleton_default();

  public:
    typedef T object_type;

    // If, at any point (in user code), singleton_default<T>::instance()
    //  is called, then the following function is instantiated.
    static object_type & instance()
    {
      // This is the object that we return a reference to.
      // It is guaranteed to be created before main() begins because of
      //  the next line.
      static object_type obj;

      // The following line does nothing else than force the instantiation
      //  of singleton_default<T>::create_object, whose constructor is
      //  called before main() begins.
      create_object.do_nothing();

      return obj;
    }
};
template <typename T>
typename singleton_default<T>::object_creator
singleton_default<T>::create_object;
```

## 使用
模板类singleton_default<T>在编译的时候会初始化create_object变量，调用instance方法，这个是在main函数之前发生的，这样也就不会存在线程竞争的问题了。

## 测试
```
 51 template <typename T>
 52 struct singleton_default
 53 {
 54   private:
 55     struct object_creator
 56     {
 57       // This constructor does nothing more than ensure that instance()
 58       //  is called before main() begins, thus creating the static
 59       //  T object before multithreading race issues can come up.
 60       object_creator() {
 61         printf("0000000000\n");
 62         singleton_default<T>::instance();
 63         printf("111111\n");
 64       }
 65       inline void do_nothing() const { printf("2222222\n");}
 66     };
 67     static object_creator create_object;
 68 
 69     singleton_default();
 70 
 71   public:
 72     typedef T object_type;
 73 
 74     // If, at any point (in user code), singleton_default<T>::instance()
 75     //  is called, then the following function is instantiated.
 76     static object_type & instance()
 77     {
 78       // This is the object that we return a reference to.
 79       // It is guaranteed to be created before main() begins because of
 80       //  the next line.
 81       printf("33333333\n");
 82       static object_type obj;
 83       printf("4444444\n");
 84 
 85       // The following line does nothing else than force the instantiation
 86       //  of singleton_default<T>::create_object, whose constructor is
 87       //  called before main() begins.
 88       create_object.do_nothing();
 89 
 90       return obj;
 91     }
 92 };
 93 template <typename T>
 94 typename singleton_default<T>::object_creator
 95 singleton_default<T>::create_object;
 96 
 97 class A {
 98 public:
 99     A() {
100         cout << "A" << endl;
101     }
102     void print() {
103         cout << "xxxx" << endl;
104     }
105 };
106 int main() {
107     printf("------------\n");
108     singleton_default<A>::instance();
109     return 0;
110 } 
```

输出：

```
0000000000
33333333
A
4444444
2222222
111111
------------
33333333
4444444
2222222
```
可以看到在main函数之前就已经调用create_object的构造函数，从而构造了class A。

c++ 初始化顺序里面看到一段话，解释了这个现象：<br/>
**All non-local variables with static storage duration are initialized as part of program startup, before the execution of the main function begins (unless deferred, see below). All variables with thread-local storage duration are initialized as part of thread launch, sequenced-before the execution of the thread function begins. <br/>
Together, zero-initialization and constant initialization are called static initialization; all other initialization is dynamic initialization. Static initialization shall be performed before any dynamic initialization takes place.**

boost库里面代码实现非常巧妙，值得好好学习。不过上面单例模式有个缺点就是T类型必须是通过默认构造函数初始化的。


## 资料
1. c++11里面static变量线程安全，c++98里面不安全
2. http://www.modernescpp.com/index.php/thread-safe-initialization-of-a-singleton 这篇文章测试各种单例模式速度