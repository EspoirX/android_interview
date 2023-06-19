Svga 文件组成：  
Svga 文件是由多个图片加一个份配置文件（json）组成，设计在制作 svga 的时候在需要变化的帧上打上动画标记，那么配置文件在该帧的地方会生成一份对应动画的数据。所以每个帧信息记录着这个 svga 图像在这个帧需要展示的形状，大小，位置等，然后在解析的时候根据这些配置信息去做该帧下图片的对应动画。

源码：
```kotlin
val parser = SVGAParser(context)
parser.decodeFromURL(URL(options.res as String), object : SVGAParser.ParseCompletion {
   //..
})
```
Svga 解析器会先调用 decodeFromURL 方法加载在线 url
```kotlin
fun decodeFromURL(
    url: URL,
    callback: ParseCompletion?,
    playCallback: PlayCallback? = null
): (() -> Unit)? {
    //...
    val urlPath = url.toString() 
    val cacheKey = SVGACache.buildCacheKey(url);
    return if (SVGACache.isCached(cacheKey)) {
        LogUtils.info(TAG, "this url cached")
        threadPoolExecutor.execute {
            if (SVGACache.isDefaultCache()) {
                this.decodeFromCacheKey(cacheKey, callback, alias = urlPath)
            } else {
                this.decodeFromSVGAFileCacheKey(cacheKey, callback, playCallback, alias = urlPath)
            }
        }
        return null
    } else {
        fileDownloader.resume(url, {
            this.decodeFromInputStream(it, cacheKey, callback, false, playCallback,alias = urlPath)
        }, {
            //...
            this.invokeErrorCallback(it, callback, alias = urlPath)
        })
    }
}
```
首先会先判断本地有没有缓存，key 是根据 url 的 md5 来的，逻辑在 buildCacheKey 里，如果有缓存，就在缓存中拿出来处理，没有则会先去下载，然后处理。
没缓存下载完后会进行解压操作，最后都会调用 decodeFromCacheKey 方法。
```kotlin
private fun decodeFromCacheKey(cacheKey: String, callback: ParseCompletion?,alias: String?) {
    //...
    try {
        val cacheDir = SVGACache.buildCacheDir(cacheKey)
        File(cacheDir, "movie.binary").takeIf { it.isFile }?.let { binaryFile ->
        //...
        FileInputStream(binaryFile).use {
                this.invokeCompleteCallback(
                    SVGAVideoEntity(
                        MovieEntity.ADAPTER.decode(it),
                        cacheDir,
                        mFrameWidth,
                        mFrameHeight
                    ),
                    callback,
                    alias
                )
            }
        }
        //...
        File(cacheDir, "movie.spec").takeIf { it.isFile }?.let { jsonFile ->
            FileInputStream(jsonFile).use { fileInputStream ->
                ByteArrayOutputStream().use { byteArrayOutputStream ->
                   //...
                    byteArrayOutputStream.toString().let {
                        JSONObject(it).let { 
                            this.invokeCompleteCallback(
                                SVGAVideoEntity(
                                    it,
                                    cacheDir,
                                    mFrameWidth,
                                    mFrameHeight
                                ),
                                callback,
                                alias
                            )
                        }
                    }
                }
            }
        } 
}
```
decodeFromCacheKey 主要是对解压包中的配置文件  movie.binary 和  movie.spec 进行读取和处理。根据命名可以猜测 movie.binary 是动画数据， movie.spec 是动画规范，而且  movie.spec 读取出来是一个 json 。
最后都会通过 invokeCompleteCallback 回调出去一个 SVGAVideoEntity 对象。那么主要是看 SVGAVideoEntity 的第一个参数。

**MovieEntity.ADAPTER.decode(it)**  
svga 解析 使用的是 pb 解析，pb 是一种格式，跟json是一个道理，但它的性能更高。那么 **ADAPTER.decode **就是使用pb解析的过程，怎么解析的可以忽略，我们看 MovieEntity 的内容有什么：
```kotlin
public final class MovieEntity extends Message<MovieEntity, MovieEntity.Builder> {
   // SVGA 格式版本号
    public final String version;
   //动画参数
    public final MovieParams params;
    //Key 是位图键名，Value 是位图文件名或二进制 PNG 数据。
    public final Map<String, ByteString> images;
    //元素列表
    public final List<SpriteEntity> sprites;
    //音频列表
    public final List<AudioEntity> audios;
    //...
}
```
通过 MovieEntity 的定义可以知道 svga 里面大概会有什么信息了。pb解析最终会解析成上面那些内容
```kotlin
public final class MovieParams extends Message<MovieParams, MovieParams.Builder> {
    //画布宽
    public final Float viewBoxWidth;
   //画布高
    public final Float viewBoxHeight;
   //动画每秒播放帧数，合法值是 [1, 2, 3, 5, 6, 10, 12, 15, 20, 30, 60] 中的任意一个。
    public final Integer fps;
   //动画总帧数
    public final Integer frames;
   //...
}
```
其中动画参数包含宽高，帧数等。
```kotlin
public final class SpriteEntity extends Message<SpriteEntity, SpriteEntity.Builder> {
    //元件所对应的位图键名, 如果 imageKey 含有 .vector 后缀，
    //该 sprite 为矢量图层 含有 .matte 后缀，该 sprite 为遮罩图层。
    public final String imageKey;
   //帧列表
    public final List<FrameEntity> frames;
    //被遮罩图层的 matteKey 对应的是其遮罩图层的 imageKey.
    public final String matteKey;
	//...
}
```
```kotlin
public final class FrameEntity extends Message<FrameEntity, FrameEntity.Builder> {
    //透明度
    public final Float alpha;
    //初始约束大小
    public final Layout layout;
   //2D 变换矩阵
    public final Transform transform;
   //遮罩路径，使用 SVG 标准 Path 绘制图案进行 Mask 遮罩。
    public final String clipPath;
   //矢量元素列表
    public final List<ShapeEntity> shapes;
    //...
}
```

然后看 SVGAVideoEntity 类：
```kotlin
constructor(entity: MovieEntity, cacheDir: File, frameWidth: Int, frameHeight: Int) {
    this.mFrameWidth = frameWidth
    this.mFrameHeight = frameHeight
    this.mCacheDir = cacheDir
    this.movieItem = entity
    entity.params?.let(this::setupByMovie)
    parserImages(entity)
    resetSprites(entity)
}

constructor(json: JSONObject, cacheDir: File, frameWidth: Int, frameHeight: Int) {
    mFrameWidth = frameWidth
    mFrameHeight = frameHeight
    mCacheDir = cacheDir
    val movieJsonObject = json.optJSONObject("movie") ?: return
    setupByJson(movieJsonObject)
    parserImages(json)
    resetSprites(json)
}

private fun setupByMovie(movieParams: MovieParams) {
    val width = (movieParams.viewBoxWidth ?: 0.0f).toDouble()
    val height = (movieParams.viewBoxHeight ?: 0.0f).toDouble()
    videoSize = SVGARect(0.0, 0.0, width, height)
    FPS = movieParams.fps ?: 20
    frames = movieParams.frames ?: 0
}

private fun setupByJson(movieObject: JSONObject) {
    movieObject.optJSONObject("viewBox")?.let { viewBoxObject ->
        val width = viewBoxObject.optDouble("width", 0.0)
        val height = viewBoxObject.optDouble("height", 0.0)
        videoSize = SVGARect(0.0, 0.0, width, height)
    }
    FPS = movieObject.optInt("fps", 20)
    frames = movieObject.optInt("frames", 0)
}
```
无论是传入 MovieEntity 还是 json，都会从中取到相关信息给 videoSize（大小），FPS 和 frames（帧数） 赋值。然后调用 parserImages 和 resetSprites，resetSprites 是给 spriteList（元素列表） 赋值.
```kotlin
private fun parserImages(json: JSONObject) {
    val imgJson = json.optJSONObject("images") ?: return
    imgJson.keys().forEach { imgKey ->
        val filePath = generateBitmapFilePath(imgJson[imgKey].toString(), imgKey)
        if (filePath.isEmpty()) {
            return
        }
        val bitmapKey = imgKey.replace(".matte", "")
        val bitmap = createBitmap(filePath)
        if (bitmap != null) {
            imageMap[bitmapKey] = bitmap
        }
    }
}
```
parserImages 是根据配置信息生成对应的 bitmap，并存在 imageMap 中。
 总体来说 ，解析器的工作就是下载解压并解析文件信息，最后将这些信息都存在 SVGAVideoEntity 里面
然后播放的时候会通过 SVGAImageView 去播放：
```kotlin
(options.imageView as SVGAImageView).setVideoItem(this)
(options.imageView as SVGAImageView).startAnimation()
```
setVideoItem 先把 SVGAVideoEntity 赋值给 SVGAImageView，然后再调用 startAnimation 方法播放。
```kotlin
fun setVideoItem(videoItem: SVGAVideoEntity?, dynamicItem: SVGADynamicEntity?) {
    if (videoItem == null) {
        setImageDrawable(null)
    } else {
        val drawable = SVGADrawable(videoItem, dynamicItem ?: SVGADynamicEntity())
        drawable.cleared = true
        setImageDrawable(drawable)
    }
}
```
setVideoItem 中会创建一个  SVGADrawable 去给 ImageView 通过 setImageDrawable 赋值。

```kotlin
private fun play(range: SVGARange?, reverse: Boolean) { 
    val drawable = getSVGADrawable() ?: return
    setupDrawable()
    mStartFrame = Math.max(0, range?.location ?: 0)
    val videoItem = drawable.videoItem
    mEndFrame = Math.min(videoItem.frames - 1, ((range?.location ?: 0) + (range?.length ?: Int.MAX_VALUE) - 1))
    val animator = ValueAnimator.ofInt(mStartFrame, mEndFrame)
    animator.interpolator = LinearInterpolator()
    animator.duration = ((mEndFrame - mStartFrame + 1) * (1000 / videoItem.FPS) / generateScale()).toLong()
    animator.repeatCount = if (loops <= 0) 99999 else loops - 1
    animator.addUpdateListener(mAnimatorUpdateListener)
    animator.addListener(mAnimatorListener)
    if (reverse) {
        animator.reverse()
    } else {
        animator.start()
    }
    mAnimator = animator
}
```
startAnimation 会调用 play 方法，这里通过属性动画遍历每一帧。 getSVGADrawable 获取的就是 setVideoItem 里面 set 进去的那个 SVGADrawable，然后遍历大小为第一帧和最后一帧，
所以关键还是看 mAnimatorUpdateListener 里面是如果更新帧的：
```kotlin
private fun onAnimatorUpdate(animator: ValueAnimator?) {
    val drawable = getSVGADrawable() ?: return
    drawable.currentFrame = animator?.animatedValue as Int
    val percentage = (drawable.currentFrame + 1).toDouble() / drawable.videoItem.frames.toDouble()
    callback?.onStep(drawable.currentFrame, percentage)
}
```
最后会调 onAnimatorUpdate 方法，所以动画的播放是通过控制 currentFrame 当前帧的变化去实现的。
```kotlin
var currentFrame = 0
internal set (value) {
    if (field == value) {
        return
    }
    field = value
    invalidateSelf()
}
```
每赋值一次就会调用 invalidateSelf 刷新一下，所以还是要看 SVGADrawable draw 方法是如何绘制当前帧的。
draw 会调用 SVGACanvasDrawer 的 drawFrame 方法进行绘制，绘制就是根据当前帧的信息去绘制了。
