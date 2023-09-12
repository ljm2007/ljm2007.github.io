---
title: Parcelable
date: 2023-09-020 14:04:43
tags: android
categories: android
top_img: https://s2.loli.net/2023/09/02/zEhOgs8dyqo6ZjM.webp
cover: https://s2.loli.net/2023/09/02/zEhOgs8dyqo6ZjM.webp
---

### 一.Parcel的简介

在介绍之前我们需要先了解Parcel是什么?Parcel翻译过来是打包的意思,其实就是包装了我们需要传输的数据,然后在Binder中传输,也就是用于跨进程传输数据

简单来说，Parcel提供了一套机制，可以将序列化之后的数据写入到一个共享内存中，其他进程通过Parcel可以从这块共享内存中读出字节流，并反序列化成对象,下图是这个过程的模型。

![img](https://s2.loli.net/2023/09/02/zEhOgs8dyqo6ZjM.webp)

> Parcel可以包含原始数据类型（用各种对应的方法写入，比如writeInt(),writeFloat()等），可以包含Parcelable对象，它还包含了一个活动的IBinder对象的引用，这个引用导致另一端接收到一个指向这个IBinder的代理IBinder。

> Parcelable通过Parcel实现了read和write的方法,从而实现序列化和反序列化,我们在源码中看一下

<img src="https://s2.loli.net/2023/09/02/CGTn9pZQYilRwLD.webp" alt="img" style="zoom:80%;" />

<img src="https://upload-images.jianshu.io/upload_images/5889165-919800e7bad91a23.png?imageMogr2/auto-orient/strip|imageView2/2/w/425/format/webp" alt="img" style="zoom:80%;" />

可以看出包含了各种各样的read和write方法,最终都是通过native方法实现

![img](https://s2.loli.net/2023/09/02/mzZVr7ltUFKxh6G.webp)

### 二. Parcelable中的三大过程介绍(序列化,反序列化,描述)

到这里,基本上关系都理清了,也明白简单的介绍和原理了,接下来在实现Parcelable之前,介绍下实现Parcelable的三大流程
 首先写一个类实现Parcelable接口,会让我们实现两个方法

![img](https:////upload-images.jianshu.io/upload_images/5889165-5bda10bee0ac8a3d.png?imageMogr2/auto-orient/strip|imageView2/2/w/965/format/webp)

要实现的方法

#### 1 描述

其中describeContents就是负责文件描述,首先看一下源码的解读

![img](https:////upload-images.jianshu.io/upload_images/5889165-19dcef6f8b585f1c.png?imageMogr2/auto-orient/strip|imageView2/2/w/1053/format/webp)

描述的源码

> 通过上面的描述可以看出,只针对一些特殊的需要描述信息的对象,需要返回1,其他情况返回0就可以

#### 2 序列化

> 我们通过writeToParcel方法实现序列化,writeToParcel返回了Parcel,所以我们可以直接调用Parcel中的write方法,基本的write方法都有,对象和集合比较特殊下面单独讲,基本的数据类型除了boolean其他都有,Boolean可以使用int或byte存储

举个例子:我们将上面的User对象实现序列化,User对象包含三个字段 age,name,isMale



```java
 /**
 * 该方法负责序列化
 * @param dest
 * @param flags
 */
@Override
public void writeToParcel(Parcel dest, int flags) {
    dest.writeInt(age);
    dest.writeString(name);
    // boolean 可以使用int或byte方式进行存储,怎么存就怎么取
    dest.writeInt(isMale ? 1 : 0);
}
```

#### 3 反序列化

> 反序列化需要定义一个CREATOR的变量,上面也说了具体的做法,这里可以直接复制Android给的例子中的,也可以自己定义一个(名字千万不能改),通过匿名内部类实现Parcelable中的Creator的接口



```csharp
/**
 * 负责反序列化
 */
public static final Creator<User> CREATOR = new Creator<User>() {
    /**
     * 从序列化后的对象中创建原始对象
     */
    @Override
    public User createFromParcel(Parcel source) {
        return new User(source);
    }

    /**
     * 创建指定长度的原始对象数组
     */
    @Override
    public User[] newArray(int size) {
        return new User[size];
    }
};

public User(Parcel parcel) {
    age = parcel.readInt();
    name = parcel.readString();
    isMale = parcel.readInt() == 1;
}
```

## 三. Parcelable的实现和使用

> 根据上面三个过程的介绍,Parcelable就写完了,就可以直接在Intent中传输了,可以自己写两个Activity传输一下数据试一下,其中一个putExtra另一个getParcelableExtra即可

这里实现Parcelable也很简单

> 1.写一个类实现Parcelable然后alt+enter 添加Parcelable所需的代码块,AndroidStudio会自动帮我们实现(这里需要注意如果其中包含对象或集合需要把对象也实现Parcelable)

![img](https:////upload-images.jianshu.io/upload_images/5889165-933096221307b5c0.png?imageMogr2/auto-orient/strip|imageView2/2/w/888/format/webp)

Paste_Image.png

### 1. Parcelable中对象和集合的处理

**如果实现Parcelable接口的对象中包含对象或者集合,那么其中的对象也要实现Parcelable接口**
 如果包含对象Author和集合:



```java
public class Book implements Parcelable {

  public int id;
  public String name;
  public boolean isCell;
  public ArrayList<String> tags;
  public Author author;
  // ***** 注意: 这里如果是集合 ,一定要初始化 *****
  public ArrayList<Author> authors = new ArrayList<>();

  public Book() {
  }
}
```

> 这里包含一个Author对象和Author集合

> 首先Author需要先实现Parcelable,实现步骤就不贴出来了,和普通的对象一样,实现三个过程

针对上面Book对象的序列化和反序列化的实现:

反序列化:

```java
//反序列化
public static final Creator<Book> CREATOR = new Creator<Book>() {
    @Override
    public Book createFromParcel(Parcel in) {
        return new Book(in);
    }

    @Override
    public Book[] newArray(int size) {
        return new Book[size];
    }
};

protected Book(Parcel in) {
    id = in.readInt();
    name = in.readString();
    isCell = in.readByte() != 0;

    tags = in.createStringArrayList();

    // 读取对象需要提供一个类加载器去读取,因为写入的时候写入了类的相关信息
    author = in.readParcelable(Author.class.getClassLoader());

    //读取集合也分为两类,对应写入的两类
    //这一类需要用相应的类加载器去获取
    //in.readList(authors, Author.class.getClassLoader()); // 对应writeList

    //这一类需要使用类的CREATOR去获取
    in.readTypedList(authors, Author.CREATOR); //对应writeTypeList
    //authors = in.createTypedArrayList(Author.CREATOR);//对应writeTypeList

    //这里获取类加载器主要有几种方式
    getClass().getClassLoader();
    Thread.currentThread().getContextClassLoader();
    Author.class.getClassLoader();
}
```

序列化:



```java
@Override
public void writeToParcel(Parcel dest, int flags) {
    //下面三种是基本类型就不多说
    dest.writeInt(id);
    dest.writeString(name);
    dest.writeByte((byte) (isCell ? 1 : 0));

    //序列化一个String的集合
    dest.writeStringList(tags);

    // 序列化对象的时候传入要序列化的对象和一个flag,
    // 这里的flag几乎都是0,除非标识当前对象需要作为返回值返回,不能立即释放资源
    dest.writeParcelable(author, flags);

    // 序列化一个对象的集合有两种方式,以下两种方式都可以

    //这些方法们把类的信息和数据都写入Parcel，以使将来能使用合适的类装载器重新构造类的实例.所以效率不高
    dest.writeList(authors);

    //这些方法不会写入类的信息，取而代之的是：读取时必须能知道数据属于哪个类并传入正确的Parcelable.Creator来创建对象
    // 而不是直接构造新对象。（更加高效的读写单个Parcelable对象的方法是：
    // 直接调用Parcelable.writeToParcel()和Parcelable.Creator.createFromParcel()）
    dest.writeTypedList(authors);
}
```

> 写入和读取集合有两种方式,
>  一种是写入类的相关信息,然后通过类加载器去读取, --> writeList |  readList
>  二是不用类相关信息,创建时传入相关类的CREATOR来创建 --> writeTypeList | readTypeList | createTypedArrayList
>  第二种效率高一些

> ##### 一定要注意如果有集合定义的时候一定要初始化 like this -->
>
> public ArrayList<Author> authors = new ArrayList<>();

### 2. Parcelable和Serializable的区别和比较

> Parcelable和Serializable都是实现序列化并且都可以用于Intent间传递数据,Serializable是Java的实现方式,可能会频繁的IO操作,所以消耗比较大,但是实现方式简单   Parcelable是Android提供的方式,效率比较高,但是实现起来复杂一些 , 二者的选取规则是:***内存序列化上选择Parcelable, 存储到设备或者网络传输上选择Serializable\***(当然Parcelable也可以但是稍显复杂)

**希望这篇文章可以帮助到需要的人,如果还有其他问题或者补充可以联系我~~~**

©著作权归作者所有,转载或内容合作请联系作者

[Android-深入探索]()

作者：MrQ_Android
链接：https://www.jianshu.com/p/32a2ec8f35ae
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。