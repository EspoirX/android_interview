Glide 除了使用内存缓存还有磁盘缓存外，其实还有很多其他缓存，它的缓存结构是多级的。
其中，比较重要之一是对象池的使用。

> **我们知道如果较短时间内频繁的新建对象是会造成内存的抖动，可能造成界面的卡顿。**
> **对象池就是为了避免这种情况发生的一种常见设计模式。**
> **它会初始化一系列的对象，当需要对象的时候不是去创建一个新的对象，而是去复用闲置的对象资源。**
> **当没有闲置资源的时候进行扩容。**


Glide 中比较重要的对象池有 LruBitmapPool 和 LruArrayPool。

# LruBitmapPool

在 GlideBuilder 的 build 方法中，创建了 LruBitmapPool：

```java
if (bitmapPool == null) {
    int size = memorySizeCalculator.getBitmapPoolSize();
    if (size > 0) {
        bitmapPool = new LruBitmapPool(size);
    } else {
        bitmapPool = new BitmapPoolAdapter();
    }
}
```

Bitmap Pool 是保证 Bitmap 回收使用管理的类，避免了 Bitmap 被大量创建造成频繁 GC 的问题。

首先通过 MemorySizeCalculator#getBitmapPoolSize 获取对象池的最大 size。MemorySizeCalculator 是一个计算器，这些大小在它的构造函数中算出来的：

```java
MemorySizeCalculator(MemorySizeCalculator.Builder builder) {
    this.context = builder.context;

    arrayPoolSize =
        isLowMemoryDevice(builder.activityManager)
        ? builder.arrayPoolSizeBytes / LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR
        : builder.arrayPoolSizeBytes;
    int maxSize =
        getMaxSize(
        builder.activityManager, builder.maxSizeMultiplier, builder.lowMemoryMaxSizeMultiplier);

    int widthPixels = builder.screenDimensions.getWidthPixels();
    int heightPixels = builder.screenDimensions.getHeightPixels();
    int screenSize = widthPixels * heightPixels * BYTES_PER_ARGB_8888_PIXEL;

    int targetBitmapPoolSize = Math.round(screenSize * builder.bitmapPoolScreens);

    int targetMemoryCacheSize = Math.round(screenSize * builder.memoryCacheScreens);
    int availableSize = maxSize - arrayPoolSize;

    if (targetMemoryCacheSize + targetBitmapPoolSize <= availableSize) {
        memoryCacheSize = targetMemoryCacheSize;
        bitmapPoolSize = targetBitmapPoolSize;
    } else {
        float part = availableSize / (builder.bitmapPoolScreens + builder.memoryCacheScreens);
        memoryCacheSize = Math.round(part * builder.memoryCacheScreens);
        bitmapPoolSize = Math.round(part * builder.bitmapPoolScreens);
    }
    //...
}
```

首先判断当前是否是低内存，判断的方法是调用 ActivityManager#isLowRamDevice 方法。然后算出 arryPoolSize，arrayPoolSizeBytes 的默认值是 4 * 1024 * 1024，即 4MB。所以，arrayPoolSize 如果是低内存的话，就是 2MB，因为 _LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR  = 2。_然后算出 maxSize，再算出 availableSize，再算出 part，最后得出 bitmapPoolSize。

在 LruBitmapPool 中，相关的具体操作是交给 LruPoolStrategy 实现，是构造函数中创建：


```java
interface LruPoolStrategy {
  void put(Bitmap bitmap);

  @Nullable
  Bitmap get(int width, int height, Bitmap.Config config);

  @Nullable
  Bitmap removeLast();

  String logBitmap(Bitmap bitmap);

  String logBitmap(int width, int height, Bitmap.Config config);

  int getSize(Bitmap bitmap);
}

public LruBitmapPool(long maxSize) {
  this(maxSize, getDefaultStrategy(), getDefaultAllowedConfigs());
}

private static LruPoolStrategy getDefaultStrategy() {
    final LruPoolStrategy strategy;
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
        strategy = new SizeConfigStrategy();
    } else {
        strategy = new AttributeStrategy();
    }
    return strategy;
}
```

根据SDK的版本有不同的实现，这里看 SizeConfigStrategy。

在 SizeConfigStrategy 中，有几个对象需要关注：

```java
private final KeyPool keyPool = new KeyPool();
private final GroupedLinkedMap<Key, Bitmap> groupedMap = new GroupedLinkedMap<>();
private final Map<Bitmap.Config, NavigableMap<Integer, Integer>> sortedSizes = new HashMap<>();
```

## KeyPool
KeyPool 继承了 BaseKeyPool，提供 get 和 offer 方法，具体实现是最大长度为 20 的 ArrayDeque。

```java
abstract class BaseKeyPool<T extends Poolable> {
  private static final int MAX_SIZE = 20;
  private final Queue<T> keyPool = Util.createQueue(MAX_SIZE);

  T get() {
    T result = keyPool.poll();
    if (result == null) {
      result = create();
    }
    return result;
  }

  public void offer(T key) {
    if (keyPool.size() < MAX_SIZE) {
      keyPool.offer(key);
    }
  }

  abstract T create();
}
```

KeyPool 的 get 是经过包装的，返回的是 Key 对象，Key 是一个封装了 size 和 Bitmap.Config 的类。

```java
static class KeyPool extends BaseKeyPool<Key> {

    public Key get(int size, Bitmap.Config config) {
        Key result = get();
        result.init(size, config);
        return result;
    }

    @Override
    protected Key create() {
        return new Key(this);
    }
}

static final class Key implements Poolable {
    @Synthetic int size;
    private Bitmap.Config config;
 	
    @VisibleForTesting
    Key(KeyPool pool, int size, Bitmap.Config config) {
        this(pool);
        init(size, config);
    }

    public void init(int size, Bitmap.Config config) {
        this.size = size;
        this.config = config;
    }
    //...
}
```

## GroupedLinkedMap
SDK 提供的 LruCache 类可以帮助实现 Lru 功能，且内部的数据结构是 LinkedHashMap。但 Glide 并没有使用现成的，而是定义了一个 GroupedLinkedMap ，其类似 LinkedHashMap ，其实体 LinedEntry 可以用于实现双向链表，并通过 GroupedLinkedMap#makeTail ，GroupedLinkedMap#makeHead 方法来改变链表头尾位置，使得具有 LRU 功能。

```java
class GroupedLinkedMap<K extends Poolable, V> {
  private final LinkedEntry<K, V> head = new LinkedEntry<>();
  private final Map<K, LinkedEntry<K, V>> keyToEntry = new HashMap<>();
  //...
}
```

head 是连表的头节点。
put：
```java
public void put(K key, V value) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
        entry = new LinkedEntry<>(key);
        // 放到链表末尾
        makeTail(entry);
        keyToEntry.put(key, entry);
    } else {
        // 不需要用到这个 key。key 是 poolable 的，offer 后会放回 pool
        key.offer();
    }
    entry.add(value);
}
```
get：
```java
public V get(K key) {
    LinkedEntry<K, V> entry = keyToEntry.get(key);
    if (entry == null) {
        entry = new LinkedEntry<>(key);
        keyToEntry.put(key, entry);
    } else {
        key.offer();
    }
    // 刚刚访问过的元素，要移到链表的开头
    makeHead(entry);
    // 如果是刚刚 new 出来的 LinkedEntry，这里会返回 null
    return entry.removeLast();
}
```
removeLast：
```java
public V removeLast() {
    // LinkedEntry 组成的是一个双向的链表，head.prev 是链表的尾节点
    LinkedEntry<K, V> last = head.prev;
    while (!last.equals(head)) {
        V removed = last.removeLast();
        if (removed != null) {
            return removed;
        } else {
            // We will clean up empty lru entries since they are likely to have been one off or
            // unusual sizes and
            // are not likely to be requested again so the gc thrash should be minimal. Doing so will
            // speed up our
            // removeLast operation in the future and prevent our linked list from growing to
            // arbitrarily large
            // sizes.
            removeEntry(last);
            keyToEntry.remove(last.key);
            last.key.offer();
        }
        last = last.prev;
    }
    return null;
}
```

## LruBitmapPool 使用

LruBitmapPool 使用在图片加载完后生成 Bitmap 的时候，在上一篇知道，图片加载完后进入到了解码阶段。而解码最终会调用 Downsampler#decode 方法实现。而 Downsampler 是在 Glide 的构造方法里面创建的，它的构造函数就传入了 LruBitmapPool 对象。看到 decode 方法里面调用的 decodeFromWrappedStreams 方法：

```java
private Bitmap decodeFromWrappedStreams(InputStream is,
                                        BitmapFactory.Options options, DownsampleStrategy downsampleStrategy,
                                        DecodeFormat decodeFormat, boolean isHardwareConfigAllowed, int requestedWidth,
                                        int requestedHeight, boolean fixBitmapToRequestedDimensions,
                                        DecodeCallbacks callbacks) throws IOException {
    long startTime = LogTime.getLogTime();

    int[] sourceDimensions = getDimensions(is, options, callbacks, bitmapPool);
    int sourceWidth = sourceDimensions[0];
    int sourceHeight = sourceDimensions[1];
    String sourceMimeType = options.outMimeType;

    // If we failed to obtain the image dimensions, we may end up with an incorrectly sized Bitmap,
    // so we want to use a mutable Bitmap type. One way this can happen is if the image header is so
    // large (10mb+) that our attempt to use inJustDecodeBounds fails and we're forced to decode the
    // full size image.
    if (sourceWidth == -1 || sourceHeight == -1) {
        isHardwareConfigAllowed = false;
    }

    int orientation = ImageHeaderParserUtils.getOrientation(parsers, is, byteArrayPool);
    int degreesToRotate = TransformationUtils.getExifOrientationDegrees(orientation);
    boolean isExifOrientationRequired = TransformationUtils.isExifOrientationRequired(orientation);

    int targetWidth = requestedWidth == Target.SIZE_ORIGINAL ? sourceWidth : requestedWidth;
    int targetHeight = requestedHeight == Target.SIZE_ORIGINAL ? sourceHeight : requestedHeight;

    ImageType imageType = ImageHeaderParserUtils.getType(parsers, is, byteArrayPool);

    calculateScaling(
        imageType,
        is,
        callbacks,
        bitmapPool,
        downsampleStrategy,
        degreesToRotate,
        sourceWidth,
        sourceHeight,
        targetWidth,
        targetHeight,
        options);
    calculateConfig(
        is,
        decodeFormat,
        isHardwareConfigAllowed,
        isExifOrientationRequired,
        options,
        targetWidth,
        targetHeight);

    boolean isKitKatOrGreater = Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT;
    // Prior to KitKat, the inBitmap size must exactly match the size of the bitmap we're decoding.
    if ((options.inSampleSize == 1 || isKitKatOrGreater) && shouldUsePool(imageType)) {
        int expectedWidth;
        int expectedHeight;
        if (sourceWidth >= 0 && sourceHeight >= 0
            && fixBitmapToRequestedDimensions && isKitKatOrGreater) {
            expectedWidth = targetWidth;
            expectedHeight = targetHeight;
        } else {
            float densityMultiplier = isScaling(options)
                ? (float) options.inTargetDensity / options.inDensity : 1f;
            int sampleSize = options.inSampleSize;
            int downsampledWidth = (int) Math.ceil(sourceWidth / (float) sampleSize);
            int downsampledHeight = (int) Math.ceil(sourceHeight / (float) sampleSize);
            expectedWidth = Math.round(downsampledWidth * densityMultiplier);
            expectedHeight = Math.round(downsampledHeight * densityMultiplier);

            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "Calculated target [" + expectedWidth + "x" + expectedHeight + "] for source"
                      + " [" + sourceWidth + "x" + sourceHeight + "]"
                      + ", sampleSize: " + sampleSize
                      + ", targetDensity: " + options.inTargetDensity
                      + ", density: " + options.inDensity
                      + ", density multiplier: " + densityMultiplier);
            }
        }
        // If this isn't an image, or BitmapFactory was unable to parse the size, width and height
        // will be -1 here.
        if (expectedWidth > 0 && expectedHeight > 0) {
            setInBitmap(options, bitmapPool, expectedWidth, expectedHeight);
        }
    }
    Bitmap downsampled = decodeStream(is, options, callbacks, bitmapPool);
    callbacks.onDecodeComplete(bitmapPool, downsampled);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logDecode(sourceWidth, sourceHeight, sourceMimeType, options, downsampled,
                  requestedWidth, requestedHeight, startTime);
    }

    Bitmap rotated = null;
    if (downsampled != null) {
        // If we scaled, the Bitmap density will be our inTargetDensity. Here we correct it back to
        // the expected density dpi.
        downsampled.setDensity(displayMetrics.densityDpi);

        rotated = TransformationUtils.rotateImageExif(bitmapPool, downsampled, orientation);
        if (!downsampled.equals(rotated)) {
            bitmapPool.put(downsampled);
        }
    }

    return rotated;
}
```

**int[] sourceDimensions = _getDimensions_(is, options, callbacks, bitmapPool);**
**
根据 InputStream 来获取图片的宽高：

```java
  private static int[] getDimensions(InputStream is, BitmapFactory.Options options,
      DecodeCallbacks decodeCallbacks, BitmapPool bitmapPool) throws IOException {
    options.inJustDecodeBounds = true;
    decodeStream(is, options, decodeCallbacks, bitmapPool);
    options.inJustDecodeBounds = false;
    return new int[] { options.outWidth, options.outHeight };
  }
```

BitmapFactory.Options 的 inJustDecodeBounds 设为 true 的话，BitmapFactory#decodeStream 或者  BitmapFactory#decodeResource_  _方法就不会生成 Bitmap 对象，而仅仅是读取该图片的尺寸和类型信息，这是减少内存占用的一个优化。

```java
private static Bitmap decodeStream(InputStream is, BitmapFactory.Options options,
      DecodeCallbacks callbacks, BitmapPool bitmapPool) throws IOException {
    if (options.inJustDecodeBounds) {
      is.mark(MARK_POSITION);
    } else {
      // Once we've read the image header, we no longer need to allow the buffer to expand in
      // size. To avoid unnecessary allocations reading image data, we fix the mark limit so that it
      // is no larger than our current buffer size here. We need to do so immediately before
      // decoding the full image to avoid having our mark limit overridden by other calls to
      // mark and reset. See issue #225.
      callbacks.onObtainBounds();
    }
    // BitmapFactory.Options out* variables are reset by most calls to decodeStream, successful or
    // otherwise, so capture here in case we log below.
    int sourceWidth = options.outWidth;
    int sourceHeight = options.outHeight;
    String outMimeType = options.outMimeType;
    final Bitmap result;
    TransformationUtils.getBitmapDrawableLock().lock();
    try {
      result = BitmapFactory.decodeStream(is, null, options);
    } catch (IllegalArgumentException e) {
      IOException bitmapAssertionException =
          newIoExceptionForInBitmapAssertion(e, sourceWidth, sourceHeight, outMimeType, options);
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(TAG, "Failed to decode with inBitmap, trying again without Bitmap re-use",
            bitmapAssertionException);
      }
      if (options.inBitmap != null) {
        try {
          is.reset();
          bitmapPool.put(options.inBitmap);
          options.inBitmap = null;
          return decodeStream(is, options, callbacks, bitmapPool);
        } catch (IOException resetException) {
          throw bitmapAssertionException;
        }
      }
      throw bitmapAssertionException;
    } finally {
      TransformationUtils.getBitmapDrawableLock().unlock();
    }

    if (options.inJustDecodeBounds) {
      is.reset();

    }
    return result;
  }
```

decodeStream 方法是把 InputStream 转换为 Bitmap。可以看到如果报 catch 的话会判断 options.inBitmap 是否为 null，不为 null 的话会把它存在 LruBitmapPool 中，然后把它置，再循环执行 decodeStream。

> **如果设置了 options.inBitmap 这个字段，Bitmap 在加载数据时可以复用这个字段所指向的 Bitmap 的内存空间。**
> **新增的这种内存复用的特性，可以优化掉因旧 Bitmap 内存释放和新 Bitmap 内存申请所带来的性能损耗。**
> **但是，内存能够复用也是有条件的。比如，在Android 4.4（API level 19）之前，只有新旧两个 Bitmap 的尺寸一样才能复用内存空间。Android 4.4 开始只要旧 Bitmap 的尺寸大于等于新的 Bitmap 就可以复用了。**


回到 decodeFromWrappedStreams 方法，看后半部分：

```java
//如果这不是图像，或者BitmapFactory无法解析大小，宽度和高度
//这里将是-1。
if (expectedWidth > 0 && expectedHeight > 0) {
    setInBitmap(options, bitmapPool, expectedWidth, expectedHeight);
}

@TargetApi(Build.VERSION_CODES.O)
private static void setInBitmap(
    BitmapFactory.Options options, BitmapPool bitmapPool, int width, int height) {
    @Nullable Bitmap.Config expectedConfig = null;
    // Avoid short circuiting, it appears to break on some devices.
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        if (options.inPreferredConfig == Config.HARDWARE) {
            return;
        }
        //在API 26上，对于某些图像，outConfig可能为null，即使图像有效，
        //也可以解码并填充outWidth / outHeight / outColorSpace（参见b / 71513049）。
        expectedConfig = options.outConfig;
    }

    if (expectedConfig == null) {
        // We're going to guess that BitmapFactory will return us the config we're requesting. This
        // isn't always the case, even though our guesses tend to be conservative and prefer configs
        // of larger sizes so that the Bitmap will fit our image anyway. If we're wrong here and the
        // config we choose is too small, our initial decode will fail, but we will retry with no
        // inBitmap which will succeed so if we're wrong here, we're less efficient but still correct.
        expectedConfig = options.inPreferredConfig;
    }
    // BitmapFactory will clear out the Bitmap before writing to it, so getDirty is safe.
    options.inBitmap = bitmapPool.getDirty(width, height, expectedConfig);
}
```

options.inBitmap 在这里赋值。

再看  decodeFromWrappedStreams 方法的最后部分：

```java
Bitmap downsampled = decodeStream(is, options, callbacks, bitmapPool);
callbacks.onDecodeComplete(bitmapPool, downsampled);

if (Log.isLoggable(TAG, Log.VERBOSE)) {
    logDecode(sourceWidth, sourceHeight, sourceMimeType, options, downsampled,
              requestedWidth, requestedHeight, startTime);
}

Bitmap rotated = null;
if (downsampled != null) {
    // If we scaled, the Bitmap density will be our inTargetDensity. Here we correct it back to
    // the expected density dpi.
    downsampled.setDensity(displayMetrics.densityDpi);

    rotated = TransformationUtils.rotateImageExif(bitmapPool, downsampled, orientation);
    if (!downsampled.equals(rotated)) {
        bitmapPool.put(downsampled);
    }
}
```

这里获取到转换的 Bitmap downsampled 后，会把它存在 LruBitmapPool 中。

# LruArrayPool
在 GlideBuilder 的 build 方法中，创建了 LruArrayPool：

```java
if (arrayPool == null) {
    arrayPool = new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
}
```

根据一开始的分析知道，getArrayPoolSizeInBytes 的大小是 4MB。

LruArrayPool 实现了 ArryPool，ArryPool 的主要作用是缓存 int[] 和 byte[]。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1567000393652-fb520a6a-38cf-4765-b9e0-b73d9c397409.png#align=left&display=inline&height=271&originHeight=179&originWidth=433&size=64326&status=done&width=655)

ArrayPool 的一个辅助类 ArrayAdapterInterface，它的作用是隔离 int[] 和 byte[] 的差别。

![image.png](https://cdn.nlark.com/yuque/0/2019/png/450005/1567000602028-b53b86dc-d94b-4f22-983c-fe542610b98a.png#align=left&display=inline&height=197&originHeight=122&originWidth=406&size=38554&status=done&width=654)

它的实现类有两个，分别是 ByteArrayAdapter 和 IntegerArrayAdapter：

```java
public final class ByteArrayAdapter implements ArrayAdapterInterface<byte[]> {
  private static final String TAG = "ByteArrayPool";

  @Override
  public String getTag() {
    return TAG;
  }

  @Override
  public int getArrayLength(byte[] array) {
    return array.length;
  }

  @Override
  public byte[] newArray(int length) {
    return new byte[length];
  }

  @Override
  public int getElementSizeInBytes() {
    return 1;
  }
}

public final class IntegerArrayAdapter implements ArrayAdapterInterface<int[]> {
  private static final String TAG = "IntegerArrayPool";

  @Override
  public String getTag() {
    return TAG;
  }

  @Override
  public int getArrayLength(int[] array) {
    return array.length;
  }

  @Override
  public int[] newArray(int length) {
    return new int[length];
  }

  @Override
  public int getElementSizeInBytes() {
    return 4;
  }
}
```

LruArrayPool 是 ArrayPool 的实现，它也使用了 GroupedLinkedMap 和 KeyPool。先看 put 方法：

```java
public synchronized <T> void put(T array) {
    @SuppressWarnings("unchecked")
    Class<T> arrayClass = (Class<T>) array.getClass();

    ArrayAdapterInterface<T> arrayAdapter = getAdapterFromType(arrayClass);
    int size = arrayAdapter.getArrayLength(array);
    int arrayBytes = size * arrayAdapter.getElementSizeInBytes();
    if (!isSmallEnoughForReuse(arrayBytes)) {
        return;
    }
    Key key = keyPool.get(size, arrayClass);

    groupedMap.put(key, array);
    NavigableMap<Integer, Integer> sizes = getSizesForAdapter(arrayClass);
    //key.size 其实就是数组长度
    Integer current = sizes.get(key.size);
    //TreeMap 里面记录的是每个 size 对应的数组有多少个
    sizes.put(key.size, current == null ? 1 : current + 1);
    currentSize += arrayBytes;
    evict();
}
```

getAdapterFromType 方法会根据 arrayClass 返回 IntegerArrayAdapter 还是 ByteArrayAdapter：

```java
private <T> ArrayAdapterInterface<T> getAdapterFromType(Class<T> arrayPoolClass) {
    ArrayAdapterInterface<?> adapter = adapters.get(arrayPoolClass);
    if (adapter == null) {
        if (arrayPoolClass.equals(int[].class)) {
            adapter = new IntegerArrayAdapter();
        } else if (arrayPoolClass.equals(byte[].class)) {
            adapter = new ByteArrayAdapter();
        } else {
            throw new IllegalArgumentException("No array pool found for: "
                                               + arrayPoolClass.getSimpleName());
        }
        adapters.put(arrayPoolClass, adapter);
    }
    return (ArrayAdapterInterface<T>) adapter;
}
```

getSizesForAdapter：


```java
private NavigableMap<Integer, Integer> getSizesForAdapter(Class<?> arrayClass) {
    //一个数组类型对应一个 TreeMap（int[]/byte[] 分别对应一个 TreeMap）
    NavigableMap<Integer, Integer> sizes = sortedSizes.get(arrayClass);
    if (sizes == null) {
        sizes = new TreeMap<>();
        sortedSizes.put(arrayClass, sizes);
    }
    return sizes;
}
```

添加一个元素到缓存里，有可能使得缓存的总量超过了最大限度，所以在最后调用 evict：

```java
private void evict() {
    evictToSize(maxSize);
}

private void evictToSize(int size) {
    while (currentSize > size) {
        Object evicted = groupedMap.removeLast();
        Preconditions.checkNotNull(evicted);
        ArrayAdapterInterface<Object> arrayAdapter = getAdapterFromObject(evicted);
        currentSize -= arrayAdapter.getArrayLength(evicted) * arrayAdapter.getElementSizeInBytes();
        decrementArrayOfSize(arrayAdapter.getArrayLength(evicted), evicted.getClass());
        if (Log.isLoggable(arrayAdapter.getTag(), Log.VERBOSE)) {
            Log.v(arrayAdapter.getTag(), "evicted: " + arrayAdapter.getArrayLength(evicted));
        }
    }
}

private void decrementArrayOfSize(int size, Class<?> arrayClass) {
    NavigableMap<Integer, Integer> sizes = getSizesForAdapter(arrayClass);
    Integer current = sizes.get(size);
    if (current == null) {
        throw new NullPointerException(
            "Tried to decrement empty size" + ", size: " + size + ", this: " + this);
    }
    if (current == 1) {
        sizes.remove(size);
    } else {
        sizes.put(size, current - 1);
    }
}
```

get 方法和 getExact 方法都是从缓存中取数组的方法，区别是，getExact 只会返回刚好符合所请求长度的数组，get 返回的数组则可能超过所请求长度。

```java
public synchronized <T> T getExact(int size, Class<T> arrayClass) {
    Key key = keyPool.get(size, arrayClass);
    return getForKey(key, arrayClass);
}

private <T> T getForKey(Key key, Class<T> arrayClass) {
    ArrayAdapterInterface<T> arrayAdapter = getAdapterFromType(arrayClass);
    T result = getArrayForKey(key);
    if (result != null) {
        currentSize -= arrayAdapter.getArrayLength(result) * arrayAdapter.getElementSizeInBytes();
        decrementArrayOfSize(arrayAdapter.getArrayLength(result), arrayClass);
    }

    if (result == null) {
        if (Log.isLoggable(arrayAdapter.getTag(), Log.VERBOSE)) {
            Log.v(arrayAdapter.getTag(), "Allocated " + key.size + " bytes");
        }
        result = arrayAdapter.newArray(key.size);
    }
    return result;
}

private <T> T getArrayForKey(Key key) {
    return (T) groupedMap.get(key);
}
```

get：


```java
public synchronized <T> T get(int size, Class<T> arrayClass) {
    // ceilingKey 的返回值大于等于 size
    // 这里就是我们要用 NavigableMap 的原因了，我们需要他的 ceilingKey()
    Integer possibleSize = getSizesForAdapter(arrayClass).ceilingKey(size);
    final Key key;
    if (mayFillRequest(size, possibleSize)) {
        key = keyPool.get(possibleSize, arrayClass);
    } else {
        key = keyPool.get(size, arrayClass);
    }
    return getForKey(key, arrayClass);
}

private boolean mayFillRequest(int requestedSize, Integer actualSize) {
    return actualSize != null
        && (isNoMoreThanHalfFull() || actualSize <= (MAX_OVER_SIZE_MULTIPLE * requestedSize));
}
```

因为缓存里可能没有我们需要的 size，但是有更大的。返回长度更大的对客户端并不会有什么影响，在某些情况下，还能增加缓存的利用率。但是，如果我们返回了一个更大的数组，就意味着有一部分空间客户端并不会使用，所以要对这个浪费的空间做一些限制。这个限制由 `MAX_OVER_SIZE_MULTIPLE` 控制。按照 4.7.1 版本的实现，这个值为 8，也就是说，可能会返回 8 倍于所请求长度的数组。

LruArrayPool 的使用在 Downsampler 的 decode 方法也有体现：

```java
public Resource<Bitmap> decode(InputStream is, int requestedWidth, int requestedHeight,
                               Options options, DecodeCallbacks callbacks) throws IOException {
    Preconditions.checkArgument(is.markSupported(), "You must provide an InputStream that supports"
                                + " mark()");

    byte[] bytesForOptions = byteArrayPool.get(ArrayPool.STANDARD_BUFFER_SIZE_BYTES, byte[].class);
    BitmapFactory.Options bitmapFactoryOptions = getDefaultOptions();
    bitmapFactoryOptions.inTempStorage = bytesForOptions;

    DecodeFormat decodeFormat = options.get(DECODE_FORMAT);
    DownsampleStrategy downsampleStrategy = options.get(DownsampleStrategy.OPTION);
    boolean fixBitmapToRequestedDimensions = options.get(FIX_BITMAP_SIZE_TO_REQUESTED_DIMENSIONS);
    boolean isHardwareConfigAllowed =
        options.get(ALLOW_HARDWARE_CONFIG) != null && options.get(ALLOW_HARDWARE_CONFIG);

    try {
        Bitmap result = decodeFromWrappedStreams(is, bitmapFactoryOptions,
                                                 downsampleStrategy, decodeFormat, isHardwareConfigAllowed, requestedWidth,
                                                 requestedHeight, fixBitmapToRequestedDimensions, callbacks);
        return BitmapResource.obtain(result, bitmapPool);
    } finally {
        releaseOptions(bitmapFactoryOptions);
        byteArrayPool.put(bytesForOptions);
    }
}
```

