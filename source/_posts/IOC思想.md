---
title: IOC思想
date: 2016-12-29 13:19:40
tags: [Android,Java,Butterknife,DI,Dagger2,IOC,Spring,xUtils]
---

### 简介：
	Spring容器来实现这些相互依赖对象的创建、协调工作。
	
### 例子：
西游记中六小龄童大闹天宫。

### 一般实现：

```
public class XiYouJi{
   public void tianGong(){
       //①演员直接侵入剧本
       LiuXiaoLingTong lxlt = new LiuXiaoLingTong();
       lxlt.action("大闹天宫！");
   }
}
```
我们会发现以上剧本在①处，作为具体角色饰演者的六小龄童直接侵入到剧本中，使剧本和演员直接耦合在一起。
![](/img/ioc1.png)

### 改善：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;一个明智的编剧在剧情创作时应围绕故事的角色进行，而不应考虑角色的具体饰演者，这样才可能在剧本投拍时自由地遴选任何适合的演员，而非绑定在六小龄童一人身上。通过以上的分析，我们知道需要为该剧本主人公孙悟空定义一个接口：
<!--more-->
```
public class XiYouJi {
   public void tianGong()
   {
        //①引入孙悟空角色接口
       SunWuKong sunWuKong = new LiuXiaoLingTong();
        //②通过接口开展剧情
        sunWuKong.action("大闹天宫！");
   }
}
```

在①处引入了剧本的角色——孙悟空，剧本的情节通过角色展开，在拍摄时角色由演员饰演，如②处所示。因此西游记、孙悟空、六小龄童三者的类图关系如图所示：

![](/img/ioc2.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;我们可以看出XiYouJi同时依赖于SunWuKong接口和LiuXiaoLingTong类，并没有达到我们所期望的剧本仅依赖于角色的目的。但是角色最终必须通过具体的演员才能完成拍摄，如何让LiuXiaoLingTong和剧本无关而又能完成SunWuKong的具体动作呢？当然是在影片投拍时，导演将LiuXiaoLingTong安排在SunWuKong的角色上，导演将剧本、角色、饰演者装配起来。

![](/img/ioc3.png)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;通过引入导演，使剧本和具体饰演者解耦了。对应到软件中，导演像是一个装配器，安排演员表演具体的角色。
现在我们可以反过来讲解IoC的概念了。IoC（Inverse of Control）的字面意思是控制反转，它包括两个内容：

* 其一是控制
* 其二是反转

### IOC的类型
#### 构造函数注入

```
public class XiYouJi {
   private SunWuKong swk;
   //①注入孙悟空的具体扮演者
   public XiYouJi(SunWuKong swk){
       this.swk = swk;
   }
    public void tianGong(){
       swk.action("大闹天宫！");
   }
}
public class Director {
   public void direct(){
        //①指定角色的扮演者
       SunWuKong swk = new LiuXiaoLingTong();
        //②注入具体扮演者到剧本中
       XiYouJi xyj = new XiYouJi(swk);
       xyj.tianGong();
   }
}
```

#### 属性注入

```
public class XiYouJi {
    private SunWuKong swk;
     //①属性注入方法
    public void setSunWuKong(SunWuKong swk) {
        this.swk = swk;
    }
    public void tianGong() {
        swk.action("大闹天宫！");
    }
}
public class Director {
   public void direct(){
       SunWuKong swk = new LiuXiaoLingTong();
       XiYouJi xyj = new XiYouJi();
        //①调用属性Setter方法注入
       xyj.setSunWuKong(swk);
       xyj.tianGong();
   }
}
```

#### 接口注入

```
public interface ActorArrangable {
   void injectSunWuKong(SunWuKong swk);
}
public class XiYouJi implements ActorArrangable {
    private SunWuKong swk;
     //①实现接口方法
    public void injectSunWuKong(SunWuKong swk) {
        this.swk = swk;
    }
    public void tianGong() {
        swk.action("大闹天宫！");
    }
}
public class Director {
   public void direct(){
       SunWuKong swk = new LiuXiaoLingTong();
       XiYouJi xyj = new XiYouJi();
       xyj.injectSunWuKong(swk);
       xyj.tianGong();
   }
}
```

#### 通过容器完成依赖关系的注入
java:spring;android:xutils（java反射和动态代理）

反射（配置文件的读取、利用反射机制注入对象）：

1. 在运行时判断任意一个对象所属的类。
2. 在运行时构造任意一个类的对象。
3. 在运行时判断任意一个类所具有的成员变量和方法。
4. 在运行时调用任意一个对象的方法

动态代理

程序运行时，允许改变程序结构或变量类型，这种语言称为动态语言。尽管在这样的定义与分类下Java不是动态语言，它却有着一个非常突出的动态相关机制：Reflection。这个字的意思是“反射、映象、倒影”，用在Java身上指的是我们可以于运行时加载、探知、使用编译期间完全未知的classes。

动态代理类

1.Interface InvocationHandler
2.Proxy

android dagger2 butterknife（java mirror编译期根据注解动态生成代码）
这里以butterknife为例。

JDK 5中提供了apt工具用来对注解进行处理。apt是一个命令行工具，与之配套的还有一套用来描述程序语义结构的Mirror API。Mirror API（com.sun.mirror.*）描述的是程序在编译时刻的静态结构。通过Mirror API可以获取到被注解的Java类型元素的信息，从而提供相应的处理逻辑。具体的处理工作交给apt工具来完成。编写注解处理器的核心是AnnotationProcessorFactory和AnnotationProcessor两个接口。后者表示的是注解处理器，而前者则是为某些注解类型创建注解处理器的工厂。

生成代码步骤

1. 自己的java处理器 extends AbstractProcessor
2. 通过查找解析注解（findAndParseTargets）形成集合Map（复杂，不贴源代码）
3. 遍历集合，通过javaoet(代码生成库，jake大神写的)生成java源码文件。

扫描源代码（源代码中的每一部分都是Element的一个特定类型，而这些信息是由Mirror获取到）

Element
类为TypeElement，变量为VariableElement，方法为ExecuteableElement