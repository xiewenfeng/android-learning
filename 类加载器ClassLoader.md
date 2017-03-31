##ClassLoader类加载器
虚拟机需要把class文件加载进来才能创建实例对象并工作，完成类加载的角色就是ClassLoader
###1. 一个Android应用有几个ClassLoader实例？
一个运行的Android应用至少有2个ClassLoader，一个是BootClassLoader(系统启动时创建的)，一个是PathClassLoader(应用启动时创建的，用于加载/data/data/packagename/apkname.apk)
###2. 双亲代理模型
创建一个ClassLoader实列，需要使用一个现有的ClassLoader实例作为新创建的实例的Parent。
```java
ClassLoader(ClassLoader parentLoader, boolean nullAllowed) {
        if (parentLoader == null && !nullAllowed) {
            throw new NullPointerException("parentLoader == null && !nullAllowed");
        }
        parent = parentLoader;
    }
```
这样一个Android应用，甚至整个Android系统里所有的ClassLoader实例都会被一棵树关联起来，这也是ClassLoader的双亲代理模型(Parent-Delegation Model)的特点。
```java
public Class<?> loadClass(String className) throws ClassNotFoundException {
        return loadClass(className, false);
    }

    protected Class<?> loadClass(String className, boolean resolve) throws ClassNotFoundException {
        //1. 先查询当前ClassLoader实例是否加载过此类，有则直接返回
        Class<?> clazz = findLoadedClass(className);

        if (clazz == null) {
            ClassNotFoundException suppressed = null;
            try {
                //2.查找父类的ClassLoader是否加载过此类，有则返回
                clazz = parent.loadClass(className, false);
            } catch (ClassNotFoundException e) {
                suppressed = e;
            }

            if (clazz == null) {
                try {
                    //若继承路线上的ClassLoader都没有加载，则由Child执行此类的加载工作
                    clazz = findClass(className);
                } catch (ClassNotFoundException e) {
                    e.addSuppressed(suppressed);
                    throw e;
                }
            }
        }

        return clazz;
    }
```
这种从继承路上查询类加载实例，如果一个类被位于树根的ClassLoader加载过，在以后的整个系统的生命周期内，这个类永远不会被重新加载。
***注意点***
在升级一些逻辑代码，通过动态加载dex文件获得新类替换原有的旧类时，为达到修复原有类的bug，就必须保证在加载新类的时候，旧类还没有被加载，若已加载过旧类，那么ClassLoader会一直优先使用旧类。
Java中判断一个类是否被加载过是通过判断
同一个Class = 相同的ClassName+PackageName+ClassLoader
所以如果使用不同的ClassLoader加载同一个类，也会造成类型不一样

###双亲代理模型的作用：
1) 共享功能：一些Framework层级的类一旦被顶层的ClassLoader加载过就缓存在内存里，以后任何地方要用，都不需要重新加载；
2) 隔离功能：不同继承路线上的ClassLoader加载的类肯定不是同一个类，这样限制避免用户自己的代码冒充核心类库的类访问枋心库包里可见的成员变量。
