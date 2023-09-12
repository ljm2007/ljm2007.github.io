---
title: BlackReflection
date: 2023-09-04 12:25:07
tags: HeiBaoBox
categories: HeiBaoBox
top_img: https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg
cover: https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg
---

<img src="https://s2.loli.net/2023/09/09/qIJnlMTRFeWhciy.jpg" alt="favicon" style="zoom:150%;" />

## **在学习HeiBaoBox时遇到一个反射的类包** 

**BlackReflection提供了一系列API来方便地使用Java Reflection。开发者可以使用注解来指定类、字段和方法。然后它会自动生成反射代码，开发人员不需要编写额外的代码来实现Java Reflection。**

github地址:https://github.com/CodingGay/BlackReflection

## 问题1 使用上还有问题

这个问题没有解决

## 问题2 在HeiBao中的使用

基本确定是反射了外部系统中的类

```java
@BClassName("android.content.pm.PackageParser") //和外部系统关联的类
public interface PackageParserMarshmallow {
    @BConstructor
    PackageParser _new();

//    @BStaticMethod
//    ActivityInfo generateActivityInfo();
//
//    @BStaticMethod
//    ApplicationInfo generateApplicationInfo();
//
//    @BStaticMethod
//    PackageInfo generatePackageInfo();
//
//    @BStaticMethod
//    ProviderInfo generateProviderInfo();
//
//    @BStaticMethod
//    ServiceInfo generateServiceInfo();

    @BMethod
    void collectCertificates(Package p, int flags);

    @BMethod
    Package parsePackage(File File0, int int1);
}
```

下面代码是根据系统的不同,返回不同PackageParser对象 用于解析,由两部份构成BR----:是反射类前缀PackageParser--:类名Marshmallow:外部系统的名 这个叫棉花糖 属于android 6 于2015年10月正式发布

```java
public static PackageParser createParser(File packageFile) {
    if (API_LEVEL >= M) {
        return BRPackageParserMarshmallow.get()._new();
    } else if (API_LEVEL >= LOLLIPOP_MR1) {
        return BRPackageParserLollipop22.get()._new(); //Android 5.0 和 5.1 以 "Lollipop" 为代号
    } else if (API_LEVEL >= LOLLIPOP) {
        return BRPackageParserLollipop.get()._new();  //Android 5.0 和 5.1 以 "Lollipop" 为代号
    }
    return null;
}
```
