---
title: JAVA动态代理实质
categories:
- JAVA
tags:
- 动态代理
---
# 动态代理介绍
不再赘述，用代码再体现下
```java
public class ClientProxy {
    private Object client;
    public ClientProxy(Object client) {
        this.client = client;
    }
    public Object create() {
        return  Proxy.newProxyInstance(client.getClass().getClassLoader(), new Class[]{client.getClass().getInterfaces()}, (proxy, method, args) -> {
           System.out.println("----- before -----"); 
           Object result = method.invoke(client, args); 
           System.out.println("----- after -----"); 
           return result; 
        });
    }
    public static void main(String[] args){
        //IClient仅有一个方法request，Client对其的实现是：()->Syetem.out.println("hello word!")
        IClient client = new ClientProxy(new Client()).create();
        client.request();
    }
}
//----
before
hello word!
after
```
从上述代码基本可以看出动态代理使用Proxy.newProxyInstance()返回被代理的类实例。在利用反射机制执行实例方法时，可以在执行前后增加自定义的逻辑（切面）。

<!--more-->
# Proxy.newProxyInstance()
既然后续业务调用的都是这玩意儿生成的对象，那么我们来看下这个的源码。这里就只摘抄核心源码。
```java
public static Object newProxyInstance(ClassLoader loader, Class<?>[] interfaces,InvocationHandler h) throws IllegalArgumentException{
    ………
    /*
     * Look up or generate the designated proxy class.
     */
    Class<?> cl = getProxyClass0(loader, intfs);
 
    /*
     * Invoke its constructor with the designated invocation handler.
     */
    
    final Constructor<?> cons = cl.getConstructor(constructorParams);
    …………
    return cons.newInstance(new Object[]{h});
}
```
通过上述代码我们可以知道，newProxyInstance主要干的事就是用getProxyClass0获取了个Class，然后用反射实例化它，最后返回给我们用。

# getProxyClass0()
其实我们用的class是getProxClass0(loader, intfs)生成的用于创建类的二进制数据。这个二进制数据可以想象为java类文件经过编译后生成的.class。这里就不分析getProxClass0怎么生成，反正我看不懂。下面我们重点关注这伙到底生成的啥。我们用下述方法将二进制数据抽成文件，并用使用jd-gui看看其内容。
```java
public static void main(String[] args) {
    FileOutputStream out = null;
    try {
        IClient client = new Client();
        IClient cProxy = (IClient) new ClientProxy(client).create();//此处的实例已经是getProxClass0 生成
        String path = "/home/magneto/桌面/$Proxy0.class";
        byte[] classFile = ProxyGenerator.generateProxyClass("$Proxy0", Client.class.getInterfaces());
 
        out = new FileOutputStream(path);
        out.write(classFile);
        out.flush();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            out.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
getProxClass0生成的$Proxy0.class文件的内容：
```java
public final class $Proxy0 extends Proxy implements IClient {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;
 
    public $Proxy0(InvocationHandler var1) throws  {
         //va1是下面super.h
         //本例中InvocationHandler接口的invoke实现是
        /** (proxy, method, args) -> { 
         * System.out.println("----- before -----");   
         * Object result = method.invoke(client, args);   
         * System.out.println("----- after -----");   
         * return result; 
         * }
         */
        super(var1);
    }
 
    public final boolean equals(Object var1) throws  {
        try {
            return (Boolean)super.h.invoke(this, m1, new Object[]{var1});
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
 
    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    public final void request() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    public final int hashCode() throws  {
        try {
            return (Integer)super.h.invoke(this, m0, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
 
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m3 = Class.forName("org.projectGaia.fileServer.proxy.inter.IClient").getMethod("request");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```
嗒哒～其实我们实际使用的类是这个。这里需要注意这几点：
1. 该类的构造函数主要就是向父类Proxy传实现InvocationHandler接口的类实例，就是ProxyGenerator.generateProxyClass中的第三个参数。
2. 这个类不但实现了接口定义的方法，同时也实现了且仅仅实现了Object的equles，toString，hashCode方法。
3. 每个方法的实现基本都是super.h.invoke(this, m3, (Object[])null)，调用InvocationHandler的invoke方法，在本例用lambda实现了这个接口方法，功能是打印“before”和“after”
4. 这也解决的我困惑已久的问题，即InvocationHandler的invoke方法中的第一个参数proxy:Object到底传入的是啥，从源码中可以知道它就是getProxClass0生成的$Proxy0.class，其实没啥大用处。 
  
打完收工。