---
title: api和接口的区别
categoriers:
  - 后端
tags:
  - 接口
abbrlink: 37274a39
date: 2023-10-06 11:10:22
---

# api和接口的区别

### 引言

在日常的学习过程中，我们会听到接口和api的这两种说法，但是并不清楚两者有什么区别，下面总结一下他人的回答以及我自己的理解。

### 接口

#### 日常生活中的接口

拿手机充电口举例，有Micro USB接口的，有type-c接口的，还有Lightning接口（ios设备），这三个接口就是一个接口定义，规定了这个充电口的形状、大小，但是你应用于这些接口在哪些品牌的手机上、怎么实现充电、是否做成快充，我都不关心，你只要符合形状、大小，你都属于这个接口。

#### 编程语言的接口

编程语言角度的接口：

指的是具体的一种语法规则（如java语言中用 interface定义接口类型）

比如有两个类都想实现同一个功能，就可以定义一个接口，然后在接口中声明一个方法（抽象方法，只是声明，并不能实现），然后让这两个类继承接口，从而通过不同的类来重写实现这个方法，实现多态，减少代码的冗余。

（同上理解：我们只关心类实现了接口中声明的方法，并不关心实际上引用的是哪个类的对象。通过传入接口的不同实现类的对象，从而在不改变调用方代码的情况向下改变程序的功能，实现多态）
### api接口

#### 应用程序角度的接口

##### 内部接口

最常见的就是开发过程中，后端开发写好了一个方法，封装成了一个接口，供前端开发人员调用。我们就可以通过在页面上的操作，来实现某个特定的功能，这种接口就属于api接口。在springcloud的组件中OpenFeign就是一个基于注解的声明式Web服务客户端，它简化了使用HTTP API的过程，开发人员可以使用该组件定义和调用api接口

```java
@FeignClient(value = "service-user")
public interface UserInfoFeignClient {

    @GetMapping("/admin/user/userInfo/inner/getById/{id}")
    public UserInfo getById(@PathVariable Long id);
}
```

上图就是定义了一个内部接口，可供开发人员调用。

##### 外部接口

同理，当我们开发某些功能需要调用第三方接口时（比如支付功能），需要调用支付宝的第三方接口来实现支付功能，这种第三方接口，也属于api接口。

### 总结

interface是在代码中使用的接口,api是提供给外部使用的程序接入点。

前者是编程语言中使用的,没有具体实现的抽象的定义

后者其实是一个已经包含了逻辑的可执行的程序,供外部使用的。

以上就是我整理的对于接口与API的理解，如果有不对的地方欢迎指出！

