---
title:  "Android Code Review怎么做"
description: 囊括大部分安卓开发容易犯的错，整理成条文
## date: add a date when publishing
---

## 空指针篇

1、字符串比较，将常量放在前面

case:
```java
static final String DOMESTIC = "1"
domestic.equals(DOMESTIC);
```

修正为：
```java
DOMESTIC.equals(domestic);
```

2、初始化你的默认值
```java 
FilterItemVM fiterVM = new FilterItemVM();
fiterVM.filterType = FilterType.ECOMOMY;
```

> 页面间对象传参在数据不完整的情况下崩溃，多数情况下是因为接收到了空的传参未处理直接调用，建议不要依赖调用方的"安全"使用。

3、返回空列表替代null

```java
List<IBULocale> getAllLocale(){
    ...
    if(localeList == null){
        return new ArrayList<IBULocacle>();
    }
}
```
> return参数在后续会被使用，如果难以保证调用的地方都安全的调用了，最好还是不要返回null。

4、使用判空的工具类
```java 
//字符换判空
android.text.TextUtils.isEmpty()
//集合判空
com.ctrip.ibu.utility.ListUtil.hasItem()|isNullOrEmpty()
```

> 使用工具类的好处是，容易养成习惯并且避免取反等逻辑错写出现

5、使用String.valueOf(Object) 替代 Object.toString()

```java
User user = ..;
Log.d("TAG",user.toString());
```

更改为：
```java
Log.d("TAG",String.valueOf(user))
```

6、你的Deeplink/Router/CMPC每个参数都应该考虑空的情况

契约类型，传入的参数较为自由，组合也比较多。即使规范好了调用方式，也可能存在非法的传入。请考虑每个参数为空的情况。

7、使用context和activity的引用时先判空

```java
network.OnSuccess(){
    if (getActivity() != null) {
        getActivity().doSomething();
    }
}
```
> activity被回收后调用容易引起内存泄漏，相对于找内存泄漏，空安全也可能让内存泄漏不Crash。

## 类型转换篇

1、注意你的String转Number的方法

如：Double.valueOf(String),String.parseDouble(String) ，传入的类型如果不是数字格式的就会出错。做好合法性校验，甚至是trycatch。

正确如下：
```java
if (gatewayTime != null) {
    try {
            double gatewayTimeDouble = Double.parseDouble(gatewayTime);
            serverCallParams.addCurrentServerCallExtraInfos("gatewayTime", gatewayTimeDouble * 1000);
        } catch (Throwable ignore) {
    }
}
```

2、不要信任系统API返回的数据。转换操作一律trycatch.

错误示例:
```java
//获取网络代理的状态
Integer.parseInt(System.getProperty("http.proxyPort"))
```
这里返回的不一定是数字，可能的原因是安卓碎片太严重，有些厂商上的api就说不定了。

3、做好类型判断后再转换

```java
if (params instanceof CoordinatorLayout.LayoutParams) {
   ((CoordinatorLayout.LayoutParams) params).setBehavior(null);
}
```

> kotlin的话使用is关键字

## 线程安全篇

1、你的单例安全吗

```java
DBHelper instance = null;
public static DBHelper getDBHelper(Context context) {
        if (instance == null) {
            instance = new DBHelper()
        }
        return instance;
    }
```

在多线程的情况下dbHelper == null就不是一定成立的了，这就可能导致创建了多个实例出来，违背了单例的本意。

> 这里可以voliate 你的dbHelper，让这个变量对其他的线程可见。或者同步整个获取单例的操作，但是在频繁调用的情况下注意性能开销。

线程安全单例示例：

```java
private static final Object mutex = new Object();

public static IBULocaleManager getInstance() {
    IBULocaleManager result = instance;
    if (result == null) {
        synchronized (mutex) {
            result = instance;
            if (result == null) {
                result = instance = new IBULocaleManager();
            }
        }
    }
    return result;
} 
```

>在getInstance()比较频繁情况下，多数是命中非空直接实例的， 如果对整个getInstance方法加锁就会影响性能，这里只在为空需要初始化对象的时候同步和校验，大大降低了对性能的影响。

2、隐藏你可以私有的方法和变量

如果你的方法不是一个线程安全的方法，且都是内部自己在调用（Java里面指在同一个类），请直接私有掉方法/变量，避免暴露出去让调用方引入了崩溃。

3、发现死锁

- 当心嵌套锁，

如下：
```java 



private static Object lock1 = new Object();
private static Object lock2 = new Object();
 
private static class Task implements Runnable {
    @Override
    public void run() {
        synchronized (lock1) {
            synchronized (lock2) {
                try {
                    lock1.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```
- 获取锁设置超时

多数情况下ReentrantLock可以替换synchronized，获取锁时设置一个超时时间，降低死锁风险。
```java
 public void run() {
    try {
        if (lock1.tryLock(1, TimeUnit.SECONDS)) {}
    }
 }
 ```
  
- 正确释放锁
```java
try{
    do...
}catch(Exception ex){
    //lock.unlock() 不要在这里释放锁，try block里面可能也没有释放，而这里只处理异常的场景
}finally{
    //这里释放锁最佳
    lock.unlock()
}
```


4、集合遍历要当心

如果你没有选择线程安全的集合类（如CopyOnWriteArrayList)，那集合操作的地方都要当心了。

```java

List<String> list = ...
class RunableA implements Runable{
    void run(){
        list.remove(new Random().next(list.size));
    }
}

class RunableB implements Runable{
    void run(){
        for(String str:list{
            result = String.valueOf(str);
            ...
        }
    }
}
```

正确写法,操作集合的时候加同步：
```java

synchronized(list){
    list.remove(new Random().next(list.size));
}

synchronized(list){
    for(String str:list{
        result = String.valueOf(str);
        ...
    }
}
```
> 为了保证锁的正确性和不必要的性能开销，最好<b>集合对象来作为对象锁</b>，避免使用this这种对象锁，匿名内部类的话锁的就可能不是一个对象了，锁也是无效的。

5、线程的取消和关闭

这是个容易被忽视但是很重要的一点，如果你的线程不能响应错误的中断或者在任务结束的时候没有正确的关闭它，那么他们一直运转下去，资源开销等也一直存在，间接引发其他问题。

- ExecuteService shutdown()

正确的stop
```java
 public void stop() throws InterruptedException {  
    try {  
        exec.shutdown(); 
        exec.awaitTermination(TIMEOUT, UNIT);  
    }  
}  
```
>需要区分shudown()和shutdownNow()的区别，如果你已经submit的task还想继续得到执行请不要使用和shutdownNow。

- "毒丸"/"标志位"中断

投放"毒丸"
```java
private volatile boolean canceled = false;
```

遇到毒丸放弃
```java
while(!canceled){
    doTask()
}
``` 
- Future cancle

```java
Future future = service.submit(new Task());
try {
   // 可能抛出异常
   future.get();
} catch (InterruptedException e) {
    e.printStackTrace();
} catch (ExecutionException e) {
    e.printStackTrace();
}finally {
    //终止任务的执行
    future.cancel(true);
}
```

## Android其他篇 

1、数据库的操作

- 确保操作的helper类是单例的并考虑了线程安全问题。
- 查询的时候尽量使用索引
- 避免DB过大，会引入较多的系统性问题

<br>
2、混淆类问题

- 接入第三方库或者底层库的时候要阅读和关注混淆规则的接入
- 反射调用的类/方法检查不能混淆，引发导致classnotfound

<br>
3、注意字符串拼接性能

避免大量字符串"+"的操作

```java
String str = "start";
for(int i =0 ;i < 100;i++){
    str = str + "trip";
}
```
> 建议使用StringBuiler/StringBuffer等完成

4、注意字符串分隔性能

避免在会被频繁调用的方法中使用以下方式分隔字符串
```java
String str = "hello-world";
String[] results = str.split("-");
```
请使用性能更好的```indexOf()+substring(from+1, to)```组合

```java
String str = "hello-world";
int from = str.indexOf('-');
int to = str.indexOf('-', from+1);
String brown = str.substring(from+1, to);
```

5、别再制造Context的内存泄漏了！

你的Context对象还在被单例或者静态对象持有吗？如下：

```java
public static LeakObject getInstance(Context context) {
    if (instance == null) {
        synchronized (LeakObject.class) {
            if (instance == null) {
                instance = new LeakObject(context);
            }
        }
    }
    return instance;
}

```

LeakObject对象的生命周期的常驻的，不同于Activity的Context的生命周期，Context已经回收了可能LeakObject还在使用。这种场景应该传入 getApplicationContext()
