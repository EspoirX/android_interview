# DexPathList

接着上篇分析 DexPathList。
源码地址：[https://github.com/EspoirX/android-source/blob/master/DexPathList.java](https://github.com/EspoirX/android-source/blob/master/DexPathList.java)

```java
/*package*/ final class DexPathList {
    private static final String DEX_SUFFIX = ".dex";
    private static final String JAR_SUFFIX = ".jar";
    private static final String ZIP_SUFFIX = ".zip";
    private static final String APK_SUFFIX = ".apk";
 
    private final ClassLoader definingContext;
    private final Element[] dexElements;
    private final File[] nativeLibraryDirectories;
 
    public DexPathList(ClassLoader definingContext, String dexPath,
            String libraryPath, File optimizedDirectory) {
        //...
        this.definingContext = definingContext;
        this.dexElements =
            makeDexElements(splitDexPath(dexPath), optimizedDirectory);
        this.nativeLibraryDirectories = splitLibraryPath(libraryPath);
    }
}
```

构造函数主要是给变量赋值。其中调用了 makeDexElements 生成一个 Element[] dexElements 数组。
Element 是 DexPathList 的一个内部类：

```java
/*package*/ static class Element {
    public final File file;
    public final ZipFile zipFile;
    public final DexFile dexFile;
    public Element(File file, ZipFile zipFile, DexFile dexFile) {
        this.file = file;
        this.zipFile = zipFile;
        this.dexFile = dexFile;
    }
    public URL findResource(String name) {
        if ((zipFile == null) || (zipFile.getEntry(name) == null)) {
            return null;
        }
        try {
            return new URL("jar:" + file.toURL() + "!/" + name);
        } catch (MalformedURLException ex) {
            throw new RuntimeException(ex);
        }
    }
}
```

makeDexElements：

```java
private static Element[] makeDexElements(ArrayList<File> files,
                                         File optimizedDirectory) {
    ArrayList<Element> elements = new ArrayList<Element>();
   
    for (File file : files) {
        ZipFile zip = null;
        DexFile dex = null;
        String name = file.getName();
        if (name.endsWith(DEX_SUFFIX)) {
            // Raw dex file (not inside a zip/jar).
            try {
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException ex) {
                System.logE("Unable to load dex file: " + file, ex);
            }
        } else if (name.endsWith(APK_SUFFIX) || name.endsWith(JAR_SUFFIX)
                   || name.endsWith(ZIP_SUFFIX)) {
            try {
                zip = new ZipFile(file);
            } catch (IOException ex) {
                System.logE("Unable to open zip file: " + file, ex);
            }
            try {
                dex = loadDexFile(file, optimizedDirectory);
            } catch (IOException ignored) {
            }
        } else {
            System.logW("Unknown file type for: " + file);
        }
        if ((zip != null) || (dex != null)) {
            elements.add(new Element(file, zip, dex));
        }
    }
    return elements.toArray(new Element[elements.size()]);
}
```

遍历 dexPath 列表，判断文件名是否是 .dex 文件，是的话通过 loadDexFile 方法给 dex 对象赋值，如果是 .apk 或者 .jar 文件，则创建 zip 对象和 dex 对象。然后根据 dex 和 zip 对象放在 Element 对象中保存并添加到 elements 中。最后通过 toArray 将 list 转为 array。

```java
private static DexFile loadDexFile(File file, File optimizedDirectory)
    throws IOException {
    if (optimizedDirectory == null) {
        return new DexFile(file);
    } else {
        String optimizedPath = optimizedPathFor(file, optimizedDirectory);
        return DexFile.loadDex(file.getPath(), optimizedPath, 0);
    }
}

static public DexFile loadDex(String sourcePathName, String outputPathName,
                              int flags) throws IOException {
    return new DexFile(sourcePathName, outputPathName, flags);
}

private static String optimizedPathFor(File path,
                                       File optimizedDirectory) {
    String fileName = path.getName();
    if (!fileName.endsWith(DEX_SUFFIX)) {
        int lastDot = fileName.lastIndexOf(".");
        if (lastDot < 0) {
            fileName += DEX_SUFFIX;
        } else {
            StringBuilder sb = new StringBuilder(lastDot + 4);
            sb.append(fileName, 0, lastDot);
            sb.append(DEX_SUFFIX);
            fileName = sb.toString();
        }
    }
    File result = new File(optimizedDirectory, fileName);
    return result.getPath();
}
```

## 加载类的过程

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

根据传入的完整的类名来加载对应的 class。遍历 dexElements 数组，调用 DexFile 的 loadClassBinaryName 查找，找到就返回。

热修复原理：**这里有关于热修复实现的一个点，就是将补丁 dex 文件放到 dexElements 数组前面，这样在加载 class 时，优先找到补丁包中的 dex 文件，加载到 class 之后就不再寻找，从而原来的 apk 文件中同名的类就不会再使用，从而达到修复的目的。**
**
loadClassBinaryName 方法在上一篇文章中说过最后是调用 native 的 defineClass 方法实现的。

至此，BaseDexClassLader 寻找 class 的路线就清晰了：

1. 当传入一个完整的类名，调用 BaseDexClassLader 的 findClass(String name) 方法 。BaseDexClassLader 的 findClass 方法会交给 DexPathList 的 findClass(String name) 方法处理。
2. 在 DexPathList 方法的内部，会遍历 dexFile ，通过 DexFile 的 dex.loadClassBinaryName(String name, ClassLoader loader) 来完成类的加载。

# DexClassLoader

DexClassLoader 继承 BaseDexClassLoader：

```java
public class DexClassLoader extends BaseDexClassLoader {
 
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```

DexClassLoader 可以加载 dex 文件以及包含 dex 的 apk 文件或 jar 文件，也支持从 SD卡进行加载，这也就意味着 DexClassLoader 可以在应用未安装的情况下加载 dex 相关文件。因此，它是热修复和插件化技术的基础。

** librarySearchPath 的意义：**
**
我们知道应用程序第一次被加载的时候，为了提高以后的启动速度和执行效率，Android 系统会对 dex 相关文件做一定程度的优化，并生成一个 ODEX 文件，此后再运行这个应用程序的时候，只要加载优化过的 ODEX 文件就行了，省去了每次都要优化的时间，而参数 optimizedDirectory 就是代表存储 ODEX 文件的路径，这个路径必须是一个内部存储路径。

# PathClassLoader

```java
public class PathClassLoader extends BaseDexClassLoader {
 
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }
    
    public PathClassLoader(String dexPath, String libraryPath,
            ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```

Android 系统使用 PathClassLoader 来加载系统类和应用程序的类，如果是加载非系统应用程序类，则会加载data/app/ 目录下的 dex 文件以及包含 dex 的 apk 文件或 jar 文件，不管是加载那种文件，最终都是要加载 dex 文件。
也就是说，在 Android 中，App 安装到手机后，apk 里面的 class.dex 中的 class 均是通过 PathClassLoader 来加载的。

- PathClassLoader 不建议开发直接使用
- PathClassLoader 继承自 BaseDexClassLoader，很明显 PathClassLoader 的方法实现都在BaseDexClassLoader 中。
- 从 PathClassLoader 的构造方法也可以看出它遵循了双亲委托模式。

PathClassLoader的构造方法有三个参数：

1. dexPath：dex 文件以及包含 dex 的 ap k文件或 jar 文件的路径集合，多个路径用文件分隔符分隔，默认文件分隔符为 ‘：’。
2. librarySearchPath：包含 C/C++ 库的路径集合，多个路径用文件分隔符分隔分割，可以为 null。
3. parent：ClassLoader 的 parent [双亲委托模式]
4. 


PathClassLoader 没有参数 optimizedDirectory，这是因为 PathClassLoader 已经默认了参数 optimizedDirectory的路径为：**/data/dalvik-cache**
**
# ClassLoader的实例化顺序

BootClassLoader 是在 Zygote 进程的入口方法中创建。
PathClassLoader 则是在 Zygote 进程创建 SystemServer 进程时创建的，查找路径为 **java.library.path**

# 自定义ClassLoader

**示例之 SD卡加载：**
**从 SD 卡中动态加载一个包含 class.dex 的 jar 文件，加载其中的类，并调用其方法

新建一个 Java 项目，包含两个文件：ISayHello.java 和 HelloAndroid.java.

```java
package com.jaeger;
public interface ISayHello {
   String say();
}

package com.jaeger;
public class HelloAndroid implements ISayHello {
   @Override
   public String say() {
       return "Hello Android";
   }
}
```

**导出 jar 包**
**
使用 SDK 目录 > platform-tools 里面的 dx 工具生成包含 class.dex 的 jar 包

将上一步生成的 sayhello.jar 放到 你的 SDK 下的 platform-tools 文件夹下，使用下面的命令生成 dex 化的 jar 文件，其中是 output 后面的sayhello_dex.jar 就是最终生成的 jar 包：

**dx --dex --output=sayhello_dex.jar sayhello.jar**

将 sayhello_dex.jar 文件拷贝到手机存储空间的根目录，不一定是内存卡。

新建一个 Android 项目，在 MainActivity 中添加如下的代码：

DexClassLoader 并不能直接加载外部存储的 .dex 文件，而是要先拷贝到内部存储里。这里的 dexPath 就是 .dex的外部存储路径，而 optimizedDirectory 则是内部路径，libraryPath 用 null 即可，parent 则是要传入当前应用的 ClassLoader，这与 ClassLoader 的“双亲代理模式”有关。
 
```java
package com.jaeger;
public interface ISayHello {
    String say();
}

public class MainActivity extends AppCompatActivity {
    private static final String TAG = "TestClassLoader";
    private TextView mTvInfo;
    private Button mBtnLoad;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        mTvInfo = (TextView) findViewById(R.id.tv_info);
        mBtnLoad = (Button) findViewById(R.id.btn_load);
        mBtnLoad.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                // 获取到包含 class.dex 的 jar 包文件
                final File jarFile =
                    new File(Environment.getExternalStorageDirectory().getPath() + File.separator + "sayhello_dex.jar");

                // 如果没有读权限,确定你在 AndroidManifest 中是否声明了读写权限
                Log.d(TAG, jarFile.canRead() + "");

                if (!jarFile.exists()) {
                    Log.e(TAG, "sayhello_dex.jar not exists");
                    return;
                }

                //]【核心！！！！】jarFile.getAbsolutePath()
                // getCodeCacheDir() 方法在 API 21 才能使用,实际测试替换成 getExternalCacheDir() 等也是可以的
                // 只要有读写权限的路径均可
                DexClassLoader dexClassLoader =
                    new DexClassLoader(jarFile.getAbsolutePath(), getExternalCacheDir().getAbsolutePath(), null, getClassLoader());
                try {
                    // 加载 HelloAndroid 类
                    Class clazz = dexClassLoader.loadClass("com.jaeger.HelloAndroid");
                    // 强转成 ISayHello, 注意 ISayHello 的包名需要和 jar 包中的
                    ISayHello iSayHello = (ISayHello) clazz.newInstance();
                    mTvInfo.setText(iSayHello.say());
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                } catch (InstantiationException e) {
                    e.printStackTrace();
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```


这里需要注意几点：

1. 因为需要从存储空间中读取 jar 文件，需要在 AndroidManifest 中声明读写权限
2. ISayHello 接口的包名必须一致
3. getCodeCacheDir() 方法在 API 21 才能使用，实际测试替换成 getExternalCacheDir() 等也是可以的。

 
