当Android系统安装一个应用的时候，有一步是对Dex进行优化，这个过程有一个专门的工具来处理，叫DexOpt。DexOpt 的执行过程是在第一次加载Dex文件的时候执行的。这个过程会生成一个ODEX文件，即Optimised Dex。执行ODex的效率会比直接执行Dex文件的效率要高很多。

但是在早期的Android系统中，DexOpt有一个问题，DexOpt会把每一个类的方法 id 检索起来，存在一个链表结构里面。但是这个链表的长度是用一个 short 类型来保存的，导致了方法id的数目不能够超过65536个。

当一个项目足够大的时候，显然这个方法数的上限是不够的。尽管在新版本的 Android 系统中，DexOpt 修复了这个问题，但是我们仍然需要对低版本的Android系统做兼容。

解决版本是一个 dex 装不下，就分成多个 dex 来装，在Android 5.0之前，为了解决这个问题，Google官方推出了这个类似于补丁一样的 support-library, MultiDex。而在 Android 5.0及以后，使用了 ART 虚拟机，原生支持从 APK 文件加载多个 dex 文件。

# MultiDex#install

```java
public static void install(Context context) {
    Log.i("MultiDex", "Installing application");
    if (IS_VM_MULTIDEX_CAPABLE) {
        Log.i("MultiDex", "VM has multidex support, MultiDex support library is disabled.");
    } else if (VERSION.SDK_INT < 4) {
        throw new RuntimeException("MultiDex installation failed. SDK " + VERSION.SDK_INT + " is unsupported. Min SDK version is " + 4 + ".");
    } else {
        try {
            ApplicationInfo applicationInfo = getApplicationInfo(context);
            if (applicationInfo == null) {
                Log.i("MultiDex", "No ApplicationInfo available, i.e. running on a test Context: MultiDex support library is disabled.");
                return;
            }

            doInstallation(context, new File(applicationInfo.sourceDir), new File(applicationInfo.dataDir), "secondary-dexes", "", true);
        } catch (Exception var2) {
            Log.e("MultiDex", "MultiDex installation failure", var2);
            throw new RuntimeException("MultiDex installation failed (" + var2.getMessage() + ").");
        }

        Log.i("MultiDex", "install done");
    }
}
```

在 install 方法中，判断如果虚拟机本身就支持加载多个dex文件，那么什么都不用做，然后获取到 ApplicationInfo。

ApplocationInfo 是应用信息，sourceDir 是**已经安装的 APK 的全路径** ，dataDir 的路径是  **/data/user/0/${packageName}/ **
**
最后执行 doInstallation 方法。

# doInstallation

```java

private static void doInstallation(Context mainContext, File sourceApk, File dataDir, String secondaryFolderName, String prefsKeyPrefix, boolean reinstallOnPatchRecoverableException) throws IOException, IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, SecurityException, ClassNotFoundException, InstantiationException {
    synchronized(installedApk) {
        if (!installedApk.contains(sourceApk)) {
            //添加 sourceApk 文件加入到一个 HasSet 中
            installedApk.add(sourceApk);
            if (VERSION.SDK_INT > 20) {
                Log.w("MultiDex", "MultiDex is not guaranteed to work in SDK version " + VERSION.SDK_INT + ": SDK version higher than " + 20 + " should be backed by " + "runtime with built-in multidex capabilty but it's not the " + "case here: java.vm.version=\"" + System.getProperty("java.vm.version") + "\"");
            }
            //获取 ClassLoader
            ClassLoader loader;
            try {
                loader = mainContext.getClassLoader();
            } catch (RuntimeException var25) {
                Log.w("MultiDex", "Failure while trying to obtain Context class loader. Must be running in test mode. Skip patching.", var25);
                return;
            }
            if (loader == null) {
                Log.e("MultiDex", "Context class loader is null. Must be running in test mode. Skip patching.");
            } else {
                //删除旧的 Dex 文件
                try {
                    clearOldDexDir(mainContext);
                } catch (Throwable var24) {
                    Log.w("MultiDex", "Something went wrong when trying to clear old MultiDex extraction, continuing without cleaning.", var24);
                }
				//获取非主 dex 文件
                File dexDir = getDexDir(mainContext, dataDir, secondaryFolderName);
                MultiDexExtractor extractor = new MultiDexExtractor(sourceApk, dexDir);
                IOException closeException = null;

                try {
                    List files = extractor.load(mainContext, prefsKeyPrefix, false);

                    try {
                        installSecondaryDexes(loader, dexDir, files);
                    } catch (IOException var26) {
                        if (!reinstallOnPatchRecoverableException) {
                            throw var26;
                        }

                        Log.w("MultiDex", "Failed to install extracted secondary dex files, retrying with forced extraction", var26);
                        files = extractor.load(mainContext, prefsKeyPrefix, true);
                        installSecondaryDexes(loader, dexDir, files);
                    }
                } finally {
                    try {
                        extractor.close();
                    } catch (IOException var23) {
                        closeException = var23;
                    }
                }

                if (closeException != null) {
                    throw closeException;
                }
            }
        }
    }
}
```

在 doInstallation 方法中 

1. 先添加应用 Apk 文件加入到一个 HasSet 中。
2. 获取 ClassLoader
3. 删除旧的 dex 文件
4. 在 getDexDir 方法中，新建一个存放dex的目录，路径是 **/data/user/0/${packageName}/code_cache/secondary-dexes**
5. 执行 MultiDexExtractor#load 方法把 APK 中的 dex 抽取到 dexDir 目录中，返回的 files 集合有可能为空，表示没有 secondaryDex。
6. 调用 installSecondaryDexes 安装 dex

# MultiDexExtractor#load 

```java
List<? extends File> load(Context context, String prefsKeyPrefix, boolean forceReload) throws IOException {
    Log.i("MultiDex", "MultiDexExtractor.load(" + this.sourceApk.getPath() + ", " + forceReload + ", " + prefsKeyPrefix + ")");
    if (!this.cacheLock.isValid()) {
        throw new IllegalStateException("MultiDexExtractor was closed");
    } else {
        List files;
        if (!forceReload && !isModified(context, this.sourceApk, this.sourceCrc, prefsKeyPrefix)) {
            try {
                //读缓存的 dex
                files = this.loadExistingExtractions(context, prefsKeyPrefix);
            } catch (IOException var6) {
                Log.w("MultiDex", "Failed to reload existing extracted secondary dex files, falling back to fresh extraction", var6);
                //读取缓存的dex失败，可能是损坏了，那就重新去解压apk读取，跟else代码块一样
                files = this.performExtractions();
                //保存标志位到sp，下次进来就走if了，不走else
                putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk), this.sourceCrc, files);
            }
        } else {
            if (forceReload) {
                Log.i("MultiDex", "Forced extraction must be performed.");
            } else {
                Log.i("MultiDex", "Detected that extraction must be performed.");
            }
			//没有缓存，解压apk读取
            files = this.performExtractions();
            //保存标志位到sp，下次进来就走if了，不走else
            putStoredApkInfo(context, prefsKeyPrefix, getTimeStamp(this.sourceApk), this.sourceCrc, files);
        }

        Log.i("MultiDex", "load found " + files.size() + " secondary dex files");
        return files;
    }
}
```

load 方法进来首先是判断是否有缓存，isModified 是判断标志位，通过 sp 文件去存储：

```java
private static boolean isModified(Context context, File archive, long currentCrc, String prefsKeyPrefix) {
    SharedPreferences prefs = getMultiDexPreferences(context);
    return prefs.getLong(prefsKeyPrefix + "timestamp", -1L) != getTimeStamp(archive) || prefs.getLong(prefsKeyPrefix + "crc", -1L) != currentCrc;
}

private static void putStoredApkInfo(Context context, String keyPrefix, long timeStamp, long crc, List<MultiDexExtractor.ExtractedDex> extractedDexes) {
    SharedPreferences prefs = getMultiDexPreferences(context);
    Editor edit = prefs.edit();
    edit.putLong(keyPrefix + "timestamp", timeStamp);
    edit.putLong(keyPrefix + "crc", crc);
    edit.putInt(keyPrefix + "dex.number", extractedDexes.size() + 1);
    int extractedDexId = 2;

    for(Iterator var10 = extractedDexes.iterator(); var10.hasNext(); ++extractedDexId) {
        MultiDexExtractor.ExtractedDex dex = (MultiDexExtractor.ExtractedDex)var10.next();
        edit.putLong(keyPrefix + "dex.crc." + extractedDexId, dex.crc);
        edit.putLong(keyPrefix + "dex.time." + extractedDexId, dex.lastModified());
    }

    edit.commit();
}
```

如果有缓存就执行 loadExistingExtractions 方法，否则执行 performExtractions 方法。

# performExtractions

```java
private List<MultiDexExtractor.ExtractedDex> performExtractions() throws IOException {
    //先确定命名格式
    String extractedFilePrefix = this.sourceApk.getName() + ".classes";
    this.clearDexDir();
    List<MultiDexExtractor.ExtractedDex> files = new ArrayList();
    ZipFile apk = new ZipFile(this.sourceApk); // apk转为zip格式

    try {
        int secondaryNumber = 2;
 		//apk已经是改为zip格式了，解压遍历zip文件，里面是dex文件，
        //名字有规律，如classes2.dex,classes3.dex
        for(ZipEntry dexFile = apk.getEntry("classes" + secondaryNumber + ".dex"); dexFile != null; dexFile = apk.getEntry("classes" + secondaryNumber + ".dex")) {
            //文件名：xxx.classes2.zip
            String fileName = extractedFilePrefix + secondaryNumber + ".zip";
            //创建这个classes2.zip文件
            MultiDexExtractor.ExtractedDex extractedFile = new MultiDexExtractor.ExtractedDex(this.dexDir, fileName);
            //添加 classes2.zip 到 files 中
            files.add(extractedFile);
            Log.i("MultiDex", "Extraction is needed for file " + extractedFile);
            int numAttempts = 0;
            boolean isExtractionSuccessful = false;

            while(numAttempts < 3 && !isExtractionSuccessful) {
                ++numAttempts;
                //这个方法是将classes1.dex文件写到压缩文件classes1.zip里去，最多重试三次
                extract(apk, dexFile, extractedFile, extractedFilePrefix);

                try {
                    extractedFile.crc = getZipCrc(extractedFile);
                    isExtractionSuccessful = true;
                } catch (IOException var18) {
                    isExtractionSuccessful = false;
                    Log.w("MultiDex", "Failed to read crc from " + extractedFile.getAbsolutePath(), var18);
                }

                Log.i("MultiDex", "Extraction " + (isExtractionSuccessful ? "succeeded" : "failed") + " '" + extractedFile.getAbsolutePath() + "': length " + extractedFile.length() + " - crc: " + extractedFile.crc);
                if (!isExtractionSuccessful) {
                    extractedFile.delete();
                    if (extractedFile.exists()) {
                        Log.w("MultiDex", "Failed to delete corrupted secondary dex '" + extractedFile.getPath() + "'");
                    }
                }
            }

            if (!isExtractionSuccessful) {
                throw new IOException("Could not create zip file " + extractedFile.getAbsolutePath() + " for secondary dex (" + secondaryNumber + ")");
            }

            ++secondaryNumber;
        }
    } finally {
        try {
            apk.close();
        } catch (IOException var17) {
            Log.w("MultiDex", "Failed to close resource", var17);
        }

    }

    return files;
}
```

该方法步骤：

1. 先确定命名格式为 apk 名字 + .classes
2. 创建 files ，ExtractedDex 是继承 File 的类，里面多了个 crs = -1L 的 long 类型变量。
3. 通过 ZipFile 把 apk 转为 zip 格式
4. 遍历 zip 文件，获取里面的 dex 文件，其中名字有规律，因为 secondaryNumber 初始值是2 ，所以名字规律如 classes2.dex，classes3.dex ..，为什么 secondaryNumber 初始值为 2？因为主 dex 是 classes1.dex。
5. 通过 ExtractedDex 创建名字为 xxx.classes2.zip 的 zip 文件，xxx 为第一步的命名规则。
6. 执行 extract 方法将 classes2.dex 文件写到压缩文件 classes2.zip 里去，通过 while 循环，最多重试三次。其中，extact 是一个读写文件的 IO 操作。
7. 最后通过 getZipCrc 方法获取到一个变量赋值给 ExtractedDex 的 crs 变量。然后改变isExtractionSuccessful 标志位代表成功。
8. 最中返回 files，里面存储的是存储了 dex 的 zip 文件。

整个过程就是 **解压apk，遍历出里面的 dex 文件，例如 class1.dex，class2.dex，然后又压缩成 class1.zip，class2.zip...，然后返回zip文件列表。**
**
**获取到 files 后会通过 putStoredApkInfo 方法缓存到 sp 文件中，而 loadExistingExtractions 方法的实现就是从 sp 文件中读取缓存。**
**
# installSecondaryDexes
获取到 flies 列表后回到 doInstallation 方法执行下一步的 installSecondaryDexes 方法，执行安装 dex 操作。

```java
private static void installSecondaryDexes(ClassLoader loader, File dexDir, List<? extends File> files) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException, SecurityException, ClassNotFoundException, InstantiationException {
    if (!files.isEmpty()) {
        if (VERSION.SDK_INT >= 19) {
            MultiDex.V19.install(loader, files, dexDir);
        } else if (VERSION.SDK_INT >= 14) {
            MultiDex.V14.install(loader, files);
        } else {
            MultiDex.V4.install(loader, files);
        }
    }
}
```

看 V19 以上的实现：

```java
static void install(ClassLoader loader, List<? extends File> additionalClassPathEntries, File optimizedDirectory) throws IllegalArgumentException, IllegalAccessException, NoSuchFieldException, InvocationTargetException, NoSuchMethodException, IOException {
  	//通过反射获取 ClassLoader 的 pathList 变量
    Field pathListField = MultiDex.findField(loader, "pathList");
    Object dexPathList = pathListField.get(loader);
    ArrayList<IOException> suppressedExceptions = new ArrayList();
    MultiDex.expandFieldArray(dexPathList, "dexElements", makeDexElements(dexPathList, new ArrayList(additionalClassPathEntries), optimizedDirectory, suppressedExceptions));
    if (suppressedExceptions.size() > 0) {
        Iterator var6 = suppressedExceptions.iterator();

        while(var6.hasNext()) {
            IOException e = (IOException)var6.next();
            Log.w("MultiDex", "Exception in makeDexElement", e);
        }

        Field suppressedExceptionsField = MultiDex.findField(dexPathList, "dexElementsSuppressedExceptions");
        IOException[] dexElementsSuppressedExceptions = (IOException[])((IOException[])suppressedExceptionsField.get(dexPathList));
        if (dexElementsSuppressedExceptions == null) {
            dexElementsSuppressedExceptions = (IOException[])suppressedExceptions.toArray(new IOException[suppressedExceptions.size()]);
        } else {
            IOException[] combined = new IOException[suppressedExceptions.size() + dexElementsSuppressedExceptions.length];
            suppressedExceptions.toArray(combined);
            System.arraycopy(dexElementsSuppressedExceptions, 0, combined, suppressedExceptions.size(), dexElementsSuppressedExceptions.length);
            dexElementsSuppressedExceptions = combined;
        }

        suppressedExceptionsField.set(dexPathList, dexElementsSuppressedExceptions);
        IOException exception = new IOException("I/O exception during makeDexElement");
        exception.initCause((Throwable)suppressedExceptions.get(0));
        throw exception;
    }
}
```

其中第二个参数就是 flies ，第三个参数就是上面说过的 **/data/user/0/${packageName}/code_cache/secondary-dexes **目录。

首先通过反射获取到 ClassLoader 的 pathList 变量，pathList，ClassLoader 是抽象类，它的一个实现是 BaseDexClassLoader，这个字段在这个类里面，它的类型是 DexPathList，主要是在 findClass 中发挥作用，在上篇文章 **ClassLoader 分析** 中说过。

接下来看这一行：

```java
MultiDex.expandFieldArray(dexPathList, 
                          "dexElements", 
                          makeDexElements(dexPathList,
                                          new ArrayList(additionalClassPathEntries),
                                          optimizedDirectory, 
                                          suppressedExceptions));
```

调用 makeDexElements 方法：

```java
private static Object[] makeDexElements(Object dexPathList, ArrayList<File> files, File optimizedDirectory, ArrayList<IOException> suppressedExceptions) throws IllegalAccessException, InvocationTargetException, NoSuchMethodException {
    Method makeDexElements = MultiDex.findMethod(dexPathList, "makeDexElements", ArrayList.class, File.class, ArrayList.class);
    return (Object[])((Object[])makeDexElements.invoke(dexPathList, files, optimizedDirectory, suppressedExceptions));
}
```

该方法的作用是通过反射找到 DexPathList 中的 创建 **Element[] dexElements;** 数组的 **makeDexElements** 方法并调用。
主要代码可以看 DexPathList 的源码：
[https://github.com/EspoirX/android-source/blob/master/DexPathList.java](https://github.com/EspoirX/android-source/blob/master/DexPathList.java)

然后再调用 expandFieldArray 方法扩展 dexElements 数组：

```java
private static void expandFieldArray(Object instance, String fieldName, Object[] extraElements) throws NoSuchFieldException, IllegalArgumentException, IllegalAccessException {
    Field jlrField = findField(instance, fieldName);
    Object[] original = (Object[])((Object[])jlrField.get(instance));
    Object[] combined = (Object[])((Object[])Array.newInstance(original.getClass().getComponentType(), original.length + extraElements.length));
    System.arraycopy(original, 0, combined, 0, original.length);
    System.arraycopy(extraElements, 0, combined, original.length, extraElements.length);
    jlrField.set(instance, combined);
}
```

就是创建一个新的数组，把原来数组内容（主dex）和要增加的内容（dex2、dex3...）拷贝进去，反射替换原来的dexElements 为新的数组，如下图

![16da4e5aaceaef08.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/450005/1571109411439-94818974-2ba1-4274-9e23-f3595ecccb1f.jpeg#align=left&display=inline&height=242&originHeight=197&originWidth=600&size=8003&status=done&width=737)
总结来说：

5.0 以下这个 **dexElements** 里面只有主 dex（可以认为是一个bug），没有dex2、dex3...，MultiDex 是怎么把dex2添加进去呢?
答案就是反射 **DexPathList** 的 **dexElements** 字段，然后把我们的 dex2 添加进去，当然，**dexElements** 里面放的是 **Element** 对象，我们只有 dex2 的**路径**，必须转换成 Element 格式才行，所以反射DexPathList 里面的 **makeDexElements** 方法，将 dex 文件转换成 Element 对象即可。

**DexPathList#makeDexElements**

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
 
> **makeDexElements 方法会判断 file 类型，上面讲 dex 提取的时候解压 apk 得到dex，然后又将 dex 压缩成 zip，压缩成 zip，就会走到第二个判断里去。仔细想想，其实 dex 不压缩成 zip，走第一个判断也没啥问题吧，那谷歌的MultiDex 为什么要将 dex 压缩成 zip 呢？**
> **在Android开发高手课中看到张绍文也提到这一点**

![16da4e5b007e3860.jpg](https://cdn.nlark.com/yuque/0/2019/jpeg/450005/1571110241745-ca1ab1ac-ae9c-4a9b-8fd6-f3f567a104cb.jpeg#align=left&display=inline&height=255&originHeight=210&originWidth=600&size=6128&status=done&width=730)
