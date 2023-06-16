
在最新的 Glide 源码里面，GlideModule 已经被废弃，被 AppGlideModule 所替换。

```java
public abstract class AppGlideModule extends LibraryGlideModule implements AppliesOptions {

  public boolean isManifestParsingEnabled() {
    return true;
  }

  @Override
  public void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder) {
    // Default empty impl.
  }
}

public abstract class LibraryGlideModule implements RegistersComponents {
  @Override
  public void registerComponents(@NonNull Context context, @NonNull Glide glide,
      @NonNull Registry registry) {
    // Default empty impl.
  }
}

@Deprecated
interface AppliesOptions {
  void applyOptions(@NonNull Context context, @NonNull GlideBuilder builder);
}
```

AppGlideModule 继承 LibraryGlideModule 并实现了 AppliesOptions，但是 AppliesOptions 已经被废弃，所以重写的 applyOptions 是 是属于 AppGlideModule 的。

AppGlideModule 的作用是给 Glide 做配置用的：
> **从 Glide 4.9.0 开始，在某些情形下必须完成必要的设置 (`setup`)。**
> **对于应用程序（application），仅当以下情形时才需要做设置：**
> - **使用一个或更多集成库**
> - **修改 Glide 的配置(`configuration`)（磁盘缓存大小/位置，内存缓存大小等）**
> - **扩展 Glide 的API。**
> 
**对于库（library），仅当库需要注册一个或多个组件时才需要做设置。**


在之前的分析中知道，Glide 是在 get 方法中最终调用 initializeGlide 方法创建实例的。在 initializeGlide 方法中有下面这些代码：

```java
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
    Context applicationContext = context.getApplicationContext();
    GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
    if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
        manifestModules = new ManifestParser(applicationContext).parse();
    }

    if (annotationGeneratedModule != null
        && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
        Set<Class<?>> excludedModuleClasses =
            annotationGeneratedModule.getExcludedModuleClasses();
        Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
        while (iterator.hasNext()) {
            com.bumptech.glide.module.GlideModule current = iterator.next();
            if (!excludedModuleClasses.contains(current.getClass())) {
                continue;
            }
            if (Log.isLoggable(TAG, Log.DEBUG)) {
                Log.d(TAG, "AppGlideModule excludes manifest GlideModule: " + current);
            }
            iterator.remove();
        }
    }

    if (Log.isLoggable(TAG, Log.DEBUG)) {
        for (com.bumptech.glide.module.GlideModule glideModule : manifestModules) {
            Log.d(TAG, "Discovered GlideModule from manifest: " + glideModule.getClass());
        }
    }

    RequestManagerRetriever.RequestManagerFactory factory =
        annotationGeneratedModule != null
        ? annotationGeneratedModule.getRequestManagerFactory() : null;
    builder.setRequestManagerFactory(factory);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
        module.applyOptions(applicationContext, builder);
    }
    if (annotationGeneratedModule != null) {
        annotationGeneratedModule.applyOptions(applicationContext, builder);
    }
    Glide glide = builder.build(applicationContext);
    for (com.bumptech.glide.module.GlideModule module : manifestModules) {
        module.registerComponents(applicationContext, glide, glide.registry);
    }
    if (annotationGeneratedModule != null) {
        annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide = glide;
}
```

首先会查找注解中是否有 AppGlideModule 的实现，然后找出来包装成 GeneratedAppGlideModule。GeneratedAppGlideModule 继承了 AppGlideModule。如果 annotationGeneratedModule 不为空或者 isManifestParsingEnabled 方法返回 true（默认返回true），就解析清单文件找出 GlideModule 并包装成 manifestModules。接下来是一个去重。再方法的后半段：

```java
//执行清单文字中注册的 GlideModule 的 applyOption 方法
for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.applyOptions(applicationContext, builder);
}
//执行注解中的 AppGlideModule 的 applyOption 方法
if (annotationGeneratedModule != null) {
    annotationGeneratedModule.applyOptions(applicationContext, builder);
}

Glide glide = builder.build(applicationContext);

//执行清单文字中注册的 GlideModule 的 registerComponents 方法
for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.registerComponents(applicationContext, glide, glide.registry);
}
//执行注解中的 AppGlideModule 的 registerComponents 方法
if (annotationGeneratedModule != null) {
    annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
}
```

可以看到 applyOption 和 registerComponents 方法的执行时机，appyOption 是在 Glide 创建之前执行的，它的参数传入的是 GlideBuilder，所以它的原理是修改 GlideBuilder 中的配置达到修改配置的效果，一般它是这么用的：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void applyOptions(Context context, GlideBuilder builder) {
    builder.setDefaultRequestOptions(
        new RequestOptions()
          .format(DecodeFormat.RGB_565)
          .disallowHardwareBitmaps());
  }
}
```

registerComponents 方法在 Glide 创建后执行，它传入的是 Glide 对象和 Registry 对象，Registry 对象是在 Glide 构造函数中创建的，它赋值注册 Glide 中的各种组件。
> 
**应用程序和库都可以注册很多组件来扩展 Glide 的功能。可用的组件包括：**
> 1. [**`ModelLoader`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/model/ModelLoaderFactory.html)**, 用于加载自定义的 Model(Url, Uri,任意的 POJO )和 Data(InputStreams, FileDescriptors)。**
> 2. [**`ResourceDecoder`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html)**, 用于对新的 Resources(Drawables, Bitmaps)或新的 Data 类型(InputStreams, FileDescriptors)进行解码。**
> 3. [**`Encoder`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/Encoder.html)**, 用于向 Glide 的磁盘缓存写 Data (InputStreams, FileDesciptors)。**
> 4. [**`ResourceTranscoder`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/resource/transcode/ResourceTranscoder.html)**，用于在不同的资源类型之间做转换，例如，从 BitmapResource 转换为 DrawableResource 。**
> 5. [**`ResourceEncoder`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/ResourceEncoder.html)**，用于向 Glide 的磁盘缓存写 Resources(BitmapResource, DrawableResource)。**
> 
**组件通过 **[**`Registry`**](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/Registry.html)** 类来注册。**
> **
**例如，添加一个 `ModelLoader` ，使其能从自定义的Model对象中创建一个 InputStream ：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Registry registry) {
    registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
  }
}
```

在一个 `GlideModule` 里可以注册很多组件。[`ModelLoader`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/model/ModelLoaderFactory.html) 和 [`ResourceDecoder`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/ResourceDecoder.html) 对于同样的参数类型还可以有多种实现。
### 
### 剖析(Anatomy)一个请求

被注册组件的集合（包括默认被 Glide 注册的和在 Module 中被注册的），会被用于定义一个加载路径集合。每个加载路径都是从提供给 [`load`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/RequestBuilder.html#load-java.lang.Object-) 方法的数据模型到 [`as`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/RequestManager.html#as-java.lang.Class-) 方法指定的资源类型的一个逐步演进的过程。一个加载路径（粗略地）由下列步骤组成:

1. 模型(Model) -> 数据(Data) (由`模型加载器(ModelLoader)`处理)
2. 数据(Data) -> 资源(Resource) (由`资源解析器(ResourceDecoder)`处理)
3. 资源(Resource) -> 转码后的资源(Transcoded Resource) (可选；由`资源转码器(ResourceTranscoder)`处理)

`编码器(Encoder)`可以在步骤2之前往Glide的磁盘缓存中写入数据。 `资源编码器(ResourceEncoder)`可以在步骤3之前网Glide的磁盘缓存写入资源。
当一个请求开始后，Glide将尝试所有从数据模型到请求的资源类型的可用路径。如果任何一个加载路径成功，这个请求就将成功。只有所有可用加载路径都失败时，这个请求才会失败。
### 
### 排序组件

在 [`Registry`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/Registry.html) 类中定义了 `prepend()` , `append()` 和 `replace()` 方法，它们可以用于设置 Glide 尝试每个 `ModelLoader` 和 `ResourceDecoder` 之间的顺序。对组件进行排序允许你注册一些只处理特定树模型的子集的组件（即只处理特定类型的Uri，或仅仅特定类型的图像格式），并可以在后面追加一个捕获所有类型的组件以处理其他情况。

##### prepend()
假如你的 `ModelLoader` 或者 `ResourceDecoder` 在某个地方失败了，这时候你想将已有的数据交由 Glide 的默认行为来处理，可以使用 `prepend()`。 `prepend()` 将确保你的 `ModelLoader` 或 `ResourceDecoder` 先于之前注册的其他组件并被首先执行。如果你的 `ModelLoader` 或者 `ResourceDecoder` 从其 `handles()` 方法中返回了一个 `false` 或失败，所有其他的 `ModelLoader` 或 `ResourceDecoder` 将以它们被注册的顺序执行，一次一个，作为一种回退方案。

##### append()
要处理新的数据类型或提供一个到 Glide 默认行为的回退，使用 `append()`。`append()` 将确保你的 `ModelLoader` 或 `ResourceDecoder` 仅在 Glide 的默认组件被尝试后才会被调用。 如果你正在尝试处理 Glide 的默认组件能处理的某些子类型 (例如一些特定的 Uri 授权或子类型)，你可能需要使用 `prepend()` 来确保 Glide 的默认组件不会在你的定制组件之前加载。

##### replace()
要完全替换 Glide 的默认行为并确保它绝不运行，请使用 `replace()`。 `replace()` 将移除所有处理给定模型和数据类的 `ModelLoaders`，并添加你的 `ModelLoader` 来代替。 `replace()` 在使用库(例如 OkHttp 或 Volley)替换掉 Glide 的网络逻辑时尤其有用，这种时候你会希望确保仅 OkHttp 或 Volley 被调用。

#### 添加一个 ModelLoader
举个例子，添加一个 `ModelLoader`，它从一个新的自定义 Model 对象中建立一个 InputStream：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.append(Photo.class, InputStream.class, new CustomModelLoader.Factory());
  }
}
```

在这里，`append()`可以被安全地使用，因为 Photo.class 是一个你的应用定制的模型对象，所以你知道 Glide 的默认行为中并没有你需要替换的东西。
相反，如果要处理一种新的 [`BaseGlideUrlLoader`](https://muyangmin.github.io/glide-docs-cn/javadocs/400/com/bumptech/glide/load/model/stream/BaseGlideUrlLoader.html) 中的 String Url类型，你应该使用 `prepend()` 以使你的 `ModelLoader` 在 Glide 对 `Strings` 的默认 `ModelLoaders` 之前运行：

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.prepend(String.class, InputStream.class, new CustomUrlModelLoader.Factory());
  }
}
```

最后，如果要完全移除和替换 Glide 对某种特定类型的默认处理，例如一个网络库，你应该使用 `replace()`:

```java
@GlideModule
public class YourAppGlideModule extends AppGlideModule {
  @Override
  public void registerComponents(Context context, Glide glide, Registry registry) {
    registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
  }
}
```

Glide文档：[https://muyangmin.github.io/glide-docs-cn/doc/configuration.html](https://muyangmin.github.io/glide-docs-cn/doc/configuration.html)
