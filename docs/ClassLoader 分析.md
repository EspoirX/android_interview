# 概述

App 启动过程：

- Launcher startActivity
- AMS startActivity
- Zygote fork 进程
- ActivityThread main()

4.1.  ActivityThread attach

4.2. handleBindApplication

4.3  **attachBaseContext**

4.4. installContentProviders

4.5. **Application onCreate**
- ActivityThread 进入loop循环
- **Activity生命周期回调，onCreate、onStart、onResume...**

 
Android Studio 编译过程：

- 打包资源文件，生成R.java文件（使用工具AAPT）
- 处理AIDL文件，生成java代码（没有AIDL则忽略）
- 编译 java 文件，生成对应.class文件（java compiler）
- .class 文件转换成dex文件（dex）
- 打包成没有签名的apk（使用工具apkbuilder）
- 使用签名工具给apk签名（使用工具Jarsigner）
- 对签名后的.apk文件进行对齐处理，不进行对齐处理不能发布到Google Market（使用工具zipalign）


- Android 应用打包成 apk 文件时，class 文件会被打包成一个或者多个 dex 文件。将一个 apk 文件后缀改成 .zip 格式解压后（也可以直接解压，apk 文件本质是个 zip 文件），里面就有 class.dex 文件，由于 Android 的 65K 问题（不要纠结是 64K 还是 65K），使用 MultiDex 就会生成多个 dex 文件。

- 当 Android 系统安装一个应用的时候，会针对不同平台对 Dex 进行优化，这个过程由一个专门的工具来处理，叫 DexOpt 。DexOpt 是在第一次加载 Dex 文件的时候执行的，该过程会生成一个 ODEX 文件，即 Optimised Dex。执行 ODEX 的效率会比直接执行 Dex 文件的效率要高很多，加快 App 的启动和响应。

- 最终在Android虚拟机上执行的并非Java 字节码，而是另一种字节码：**dex 格式的字节码**。在编译 Java 代码之 后 ，通过 Android 平台上的工具可以将 Java 字节码转换成 Dex 字节码。

总之，Android 中的 Dalvik/ART **无法像 JVM 那样 直接 加载 class 文件和 jar 文件中的 class**，需要通过 dx 工具来优化转换成 Dalvik byte code 才行，只能通过 **dex 或者 包含 dex 的jar、apk 文件来加载**（注意 odex 文件后缀可能是 .dex 或 .odex，也属于 dex 文件），因此 Android 中的 ClassLoader 工作就**主要交给了 BaseDexClassLoader **来处理

```java
public abstract class ClassLoader {
	//...
}

/**
 * Base class for common functionality between various dex-based
 * {@link ClassLoader} implementations.
 */
public class BaseDexClassLoader extends ClassLoader {
	//...
}
```

# ClassLoader 类型
Android中的ClassLoader类型和Java中的ClassLoader类型类似，也分为两种类型:

- 系统内置
   - **BootClassLoader**
   - **PathClassLoader**
   - **DexClassLoader**
- 用户自定义
   - **...**

![classloader.png](https://s2.loli.net/2023/06/19/xEyzfYXvGMC25qi.png)

运行一个 Android 程序需要用到几种 ClassLoader 类型？

```java
ClassLoader loader = MainActivity.class.getClassLoader();
while (loader != null) {
    LogUtil.i("xian", "loader = " + loader.toString());
    loader = loader.getParent();
}
```

输出：

```java
loader = dalvik.system.PathClassLoader[
    DexPathList[
        [zip file "/data/app/com.example.myapplication-qVQK8yfM_dQjDkRYVpZizg==/base.apk"],
         nativeLibraryDirectories=[
             /data/app/com.example.myapplication-qVQK8yfM_dQjDkRYVpZizg==/lib/arm64, 
             /data/app/com.example.myapplication-qVQK8yfM_dQjDkRYVpZizg==/base.apk!/lib/arm64-v8a, 
             /system/lib64, /product/lib64
            ]
        ]
]
loader = java.lang.BootClassLoader@bb9a922
```

1. 一种是 **PathClassLoader**. 在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的，DexPathList中包含了很多apk的路径，其中 /data/app/com.example.myapplication-qVQK8yfM_dQjDkRYVpZizg==/base.apk 就是示例应用安装在手机上的位置。关于DexPathList后续文章会进行介绍
2. 另一种则是 **BootClassLoader**。

# ClassLoader的继承关系

![classloader1.jpeg](https://s2.loli.net/2023/06/19/Q72TbneiSRfpEv1.png)

- **ClassLoader** 是一个抽象类，其中定义了ClassLoader的主要功能。
- **BootClassLoader** 是它的内部类，用于**预加载preload()**常用类，加载一些系统Framework层级需要的类，我们的Android应用里也需要用到一些系统的类等
- **SecureClassLoader** 类和JDK8中的SecureClassLoader类的代码是一样的，它继承了抽象类ClassLoader。SecureClassLoader并不是ClassLoader的实现类，而是拓展了ClassLoader类加入了权限方面的功能，加强了ClassLoader的安全性。
   - **URLClassLoader** 类和JDK8中的URLClassLoader类的代码是一样的，它继承自SecureClassLoader，用来通过URl路径从jar文件和文件夹中加载类和资源。**在Android中基本无法使用**
- **BaseDexClassLoader** 继承自ClassLoader，是抽象类ClassLoader的具体实现类，PathClassLoader和DexClassLoader都继承它。
   - **PathClassLoader** 加载系统类和应用程序的类，如果是加载非系统应用程序类，则会加载data/app/目录下的dex文件以及包含dex的apk文件或jar文件
   - **DexClassLoader** 可以加载自定义的dex文件以及包含dex的apk文件或jar文件，也支持从SD卡进行加载
   - **InMemoryDexClassLoader** 是Android8.0新增的类加载器，继承自BaseDexClassLoader，用于加载内存中的dex文件。

# ClassLoader构造函数

```java
public abstract class ClassLoader {
    static private class SystemClassLoader {
        public static ClassLoader loader = ClassLoader.createSystemClassLoader();
    }
    
	protected ClassLoader(ClassLoader parent) {
        this(checkCreateClassLoader(), parent);
    }
    
    protected ClassLoader() {
        this(checkCreateClassLoader(), getSystemClassLoader());
    }
 
    public static ClassLoader getSystemClassLoader() {
        return SystemClassLoader.loader;
    }

    private static ClassLoader createSystemClassLoader() {
       String classPath = System.getProperty("java.class.path", ".");
       String librarySearchPath = System.getProperty("java.library.path", "");
       return new PathClassLoader(classPath, librarySearchPath, BootClassLoader.getInstance());
    }
}
```

可以看到 ClassLoader 有两个构造函数。一个显式传入一个 **父构造器实例**，一个是使用 **默认父类构造器。**
 
而默认的构造器 SystemClassLoader 是 PathClassLoader，而 PathClassLoader 的默认构造器是 BootClassLoader。

# loadClass与双亲委托

```java
protected Class<?> loadClass(String name, boolean resolve)
    throws ClassNotFoundException
    {
            // First, check if the class has already been loaded
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    if (parent != null) {
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    c = findClass(name);
                }
            }
            return c;
    }
```

ClassLoader 中重要的方法是 **loadClass(String name)**，其他的子类都继承了此方法且没有进行复写。
loadClass 的逻辑是，先调用 findLoadedClass 判断这个类是否之前被加载过，如果加载过了，则直接返回，如果没加载过，则尝试调用 parent.loadClass 加载，如果加载不成功才调用 findClass 自己进行加载。

```java
protected final Class<?> findLoadedClass(String name) {
    ClassLoader loader;
    if (this == BootClassLoader.getInstance())
        loader = null;
    else
        loader = this;
    return VMClassLoader.findLoadedClass(loader, name);
}
//VMClassLoader#findLoadedClass
native static Class findLoadedClass(ClassLoader cl, String name);

//不同子类复写此方法，实现不同路径的查找  
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```

## getResource 的双亲委托

```java
public URL getResource(String name) {
    URL url;
    if (parent != null) {
        url = parent.getResource(name);
    } else {
        url = getBootstrapResource(name);
    }
    if (url == null) {
        url = findResource(name);
    }
    return url;
}

protected URL findResource(String name) {
    return null;
}
```

# BootClassLoader

```java
class BootClassLoader extends ClassLoader {

    private static BootClassLoader instance;

    @FindBugsSuppressWarnings("DP_CREATE_CLASSLOADER_INSIDE_DO_PRIVILEGED")
    public static synchronized BootClassLoader getInstance() {
        if (instance == null) {
            instance = new BootClassLoader();
        }
        return instance;
    }

    public BootClassLoader() {
        super(null);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        return Class.classForName(name, false, null);
    }

    @Override
    protected URL findResource(String name) {
        return VMClassLoader.getResource(name);
    }

    @SuppressWarnings("unused")
    @Override
    protected Enumeration<URL> findResources(String resName) throws IOException {
        return Collections.enumeration(VMClassLoader.getResources(resName));
    }
 
    @Override
    protected Package getPackage(String name) {
        if (name != null && !name.isEmpty()) {
            synchronized (this) {
                Package pack = super.getPackage(name);
                if (pack == null) {
                    pack = definePackage(name, "Unknown", "0.0", "Unknown", "Unknown", "0.0",
                            "Unknown", null);
                }
                return pack;
            }
        }
        return null;
    }

    @Override
    public URL getResource(String resName) {
        return findResource(resName);
    }

    @Override
    protected Class<?> loadClass(String className, boolean resolve)
           throws ClassNotFoundException {
        Class<?> clazz = findLoadedClass(className);
        if (clazz == null) {
            clazz = findClass(className);
        }
        return clazz;
    }

    @Override
    public Enumeration<URL> getResources(String resName) throws IOException {
        return findResources(resName);
    }
}
```

BootClassLoader 是 ClassLoader 的内部类，实现了单例模式，它的 loadClass 实现是先在父类找，然后再调用自己的 findClass 找，而 findClass 的实现是调用 Class.classForName(name, false, null);，而这个方法是 native 方法。

```java
static native Class<?> classForName(String className, boolean shouldInitialize,
                                    ClassLoader classLoader) throws ClassNotFoundException;
```


# BaseDexClassLoader
源码地址：[https://github.com/EspoirX/android-source/blob/master/BaseDexClassLoader.java](https://github.com/EspoirX/android-source/blob/master/BaseDexClassLoader.java)
```java
public class BaseDexClassLoader extends ClassLoader {

    private final String originalPath;
    private final DexPathList pathList;
    
    public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.originalPath = dexPath;
        this.pathList =
            new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
    //...
}
```

BaseDexClassLoader 构造函数有四个参数。PathClassLoader和DexClassLoader都继承自BaseDexClassLoader。

**dexPath**
指目标类所在的APK或jar文件的路径。类装载器将从该路径中寻找指定的目标类，该类必须是 APK 或 jar 的全路径。  
如果要包含多个路径,路径之间必须使用特定的分割符分隔，特定的分割符可以使用System.getProperty(“path.separtor”) 获得。

**optimizedDirectory**
optimizedDirectory 是用来缓存我们需要加载的 dex 文件的，并创建一个 DexFile 对象，如果它为 null，那么会直接使用 dex 文件原有的路径来创建 DexFile 对象。

由于 dex 文件被包含在 APK 或者 Jar 文件中，因此在装载目标类之前需要先从 APK 或 Jar 文件中解压出 dex 文件，该参数就是制定解压出的 dex 文件存放的路径。

这也是对 apk 中 dex 根据平台进行 ODEX 优化的过程。其实 APK 是一个程序压缩包，里面包含 dex 文件，ODEX 优化就是把包里面的执行程序提取出来，就变成 ODEX 文件，因为你提取出来了，系统第一次启动的时候就不用去解压程序压缩包的程序，少了一个解压的过程。这样的话系统启动就加快了。

为什么说是第一次呢？是因为 DEX 版本的也只有第一次会解压执行程序到  /data/dalvik-cache（针对PathClassLoader）或者 optimizedDirectory(针对DexClassLoader）目录，之后也是直接读取目录下的的 dex 文件，所以第二次启动就和正常的差不多了。当然这只是简单的理解，实际生成的 ODEX 还有一定的优化作用。

**无论哪种动态加载，ClassLoader 只能加载内部存储路径中的 dex 文件，所以这个路径必须为内部路径。**

**libraryPath**  
指目标类中所使用的 C/C++ 库存放的路径。

**parent**  
是指该装载器的父装载器。

# ClassLoader加载class的过程

BaseDexClassLoader 的相关操作都是委托 DexPathList 执行。

DexPathList 源码地址：[https://github.com/EspoirX/android-source/blob/master/DexPathList.java](https://github.com/EspoirX/android-source/blob/master/DexPathList.java)

BaseDexClassLoader 中有个 pathList 对象，pathList 中包含一个 **DexFile 的数组 dexElements**

- dexElements 数组就是 odex 文件的集合

odex 文件是 dexPath 指向的原始 dex(.apk,.zip,.jar等) 文件在 optimizedDirectory 文件夹中生成相应的优化后的文件。  
如果不分包，一般这个数组只有一个 Element 元素，也就只有一个 DexFile 文件。

对于类加载，就是遍历这个集合，通过 DexFile 去寻找，并最终调用 native 方法的 defineClass。

BaseDexClassLoader 的 findClass:

```java
@Override
protected Class<?> findClass(String name) throws ClassNotFoundException {
    Class clazz = pathList.findClass(name);
    if (clazz == null) {
        throw new ClassNotFoundException(name);
    }
    return clazz;
}
```

DexPathList#findClass

```java
public Class findClass(String name) {
    for (Element element : dexElements) {
        DexFile dex = element.dexFile;
        if (dex != null) {
            Class clazz = dex.loadClassBinaryName(name, definingContext);
            if (clazz != null) {
                return clazz;
            }
        }
    }
    return null;
}
```

调用的是 DexFile 的 loadClassBinaryName 方法。  
DexFile 源码地址：[https://github.com/EspoirX/android-source/blob/master/DexFile.java](https://github.com/EspoirX/android-source/blob/master/DexFile.java)

```java
public Class loadClassBinaryName(String name, ClassLoader loader) {
    return defineClass(name, loader, mCookie);
}

private native static Class defineClass(String name, ClassLoader loader, int cookie);
```

