---
title: "SimpleDateFormat的线程安全问题以及四种解决办法"
author: ""
type: ""
date: 2024-04-14T14:10:45+08:00
subtitle: ""
image: ""
tags: ["技术", "生活", "java"]
share:

---




# SimpleDateFormat的线程安全问题以及四种解决办法


### 1.多线程环境下SimpleDateFormat的不安全问题:

SimpleDateFormat的format方法实际操作的就是Calendar（Calendar变量也就是一个共享变量线程不安全）。

也正是因为每次在转化时间的时候foramat会先把时间set到calendar中，这样就会导致A线程读取到B线程的时间

![img](https://upload-images.jianshu.io/upload_images/15646494-e27eb6e4df7826db.png?imageMogr2/auto-orient/strip|imageView2/2/w/718/format/webp)

image

![img](https://upload-images.jianshu.io/upload_images/15646494-a0f547b5eb446cfd.png?imageMogr2/auto-orient/strip|imageView2/2/w/837/format/webp)

image

#### 我们来试一下：

定义两个全局常量



```cpp
private static final String myDateStr = "2022-01-01";

private static final SimpleDateFormat dateFormat = new SimpleDateFormat("yyyy-MM-dd");

//写一个测试方法：

private static void test(Callable task) throws Exception {

    ExecutorService pool = Executors.newFixedThreadPool(10);

    List<Future> list = new ArrayList<>();

    for (int i = 0; i < 10; i++) {

        Future future = pool.submit(task);

        list.add(future);

}

    for (Future future : list) {

        System.out.println(future.get());

}

    pool.shutdown();

}

//开始测试：
public static void main(String[] args) throws Exception {

    test(()->dateFormat.parse(myDateStr));

} 
```

果然控制台报错！

![img](https://upload-images.jianshu.io/upload_images/15646494-85c6942ffae23511.png?imageMogr2/auto-orient/strip|imageView2/2/w/947/format/webp)

image

#### 解决办法：

###### 1.每次使用parse直接new一个SimpleDateFormat



```java
public static void main(String[] args) throws Exception {

 test(()->new SimpleDateFormat("yyyy-MM-dd").parse(myDateStr));

} 
```

打印正常！

###### 2.使用synchronized同步锁



```java
public static void main(String[] args) throws Exception {

test(()->{ synchronized(dateFormat) { return dateFormat.parse(myDateStr);} });

} 
```

打印正常！

###### 3.使用TreadLocal



```java
private static final ThreadLocal<SimpleDateFormat> df = ThreadLocal.withInitial(() ->new SimpleDateFormat("yyyy-MM-dd"));

public static Date parseToDate(String dateString)  {

    try {

        return df.get().parse(dateString);

    } catch (ParseException e) {

        return null;

}

}

public static void main(String[] args) throws Exception {

test(()->parseToDate(myDateStr));

} 
```

打印正常！

###### 4.使用java8 DateTimeFormatter线程安全对象



```java
public static void main(String[] args) throws Exception {

test(()->LocalDate.parse(myDateStr, DateTimeFormatter.ofPattern("yyyy-MM-dd")));

} 
```

打印正常！