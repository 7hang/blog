# iOS图片内存管理和性能优化

## 图片在计算机中如何存储和表示

### 常见的图片格式

**JPEG** 是目前最常见的图片格式，它诞生于1992年，是一个很古老的格式。它只支持`有损压缩`，其压缩算法可以精确控制压缩比，以图像质量换得存储空间。由于它太过常见，以至于许多移动设备的 CPU 都支持针对它的硬编码与硬解码。

**PNG** 诞生在 1995 年，比 JPEG 晚几年。它本身的设计目的是替代 GIF 格式，所以它与 GIF 有更多相似的地方。PNG 只支持`无损压缩`，所以它的压缩比是有上限的。相对于 JPEG 和 GIF 来说，它最大的优势在于支持完整的`透明通道`。

**GIF** 诞生于 1987 年，随着初代互联网流行开来。它有很多缺点，比如通常情况下只支持 256 种颜色、透明通道只有 1 bit、文件压缩比不高。它唯一的优势就是支持`多帧动画`，凭借这个特性，它得以从 Windows 1.0 时代流行至今，而且仍然大受欢迎。

| 格式 |             优点             |                                   缺点 |             用途 |
| ---- | :--------------------------: | -------------------------------------: | ---------------: |
| jpg  |       色彩丰富，文件小       |                               有损压缩 |     颜色丰富的图 |
| png  | 透明、无损压缩、简单图文件小 | 若颜色较多复杂，则图片生成后的文件很大 | 小图标、透明背景 |
| gif  |      动态、透明、文件小      |                色域不广、只有256种颜色 |         动态图片 |

**WebP** 是 Google 在 2010 年发布的图片格式，希望以`更高的压缩比`替代 JPEG。它用 VP8 视频帧内编码作为其算法基础，取得了不错的压缩效果。它支持`有损和无损压缩`、支持完整的`透明通道`、也支持`多帧动画`，并且没有版权问题，是一种非常理想的图片格式`(美中不足的是，WebP格式图像的编码时间“比JPEG格式图像长8倍)`。借由 Google 在网络世界的影响力，WebP 在几年的时间内已经得到了广泛的应用。看看你手机里的 App：微博、微信、QQ、淘宝、网易新闻等等，每个 App 里都有 WebP 的身影。Facebook 则更进一步，用 WebP 来显示聊天界面的贴纸动画。

> 关于以上几种图片格式在移动端的解码和性能对比参见：[移动端图片格式调研](https://links.jianshu.com/go?to=https%3A%2F%2Fblog.ibireme.com%2F2015%2F11%2F02%2Fmobile_image_benchmark%2F)

## iOS中图片加载过程和性能瓶颈

如上文所说，大部分格式的图片都是被压缩的，都需要被首先解码为bitmap(未压缩的位图)，然后才能渲染到UI上。
`UIImageView` 显示图片，也有类似的过程。实际上，一张图片从在文件系统中，到被显示到 `UIImageView`，会经历以下几个步骤：

1. 假设我们使用 `+imageWithContentsOfFile:` 方法从磁盘中加载一张图片，这个时候的图片并没有解压缩；

2. 然后将生成的 `UIImage` 赋值给 `UIImageView` ；

3. 接着一个隐式的 `CATransaction` 捕获到了 `UIImageView` 图层树的变化；

4. 在主线程的下一个 `run loop` 到来时，`Core Animation` 提交了这个隐式的 `transaction`，这个过程可能会对图片进行 

   `copy` 操作，而受图片是否字节对齐等因素的影响，这个 copy 操作可能会涉及以下部分或全部步骤：

   1. 分配内存缓冲区用于管理文件 IO 和解压缩操作；
   2. 将文件数据从磁盘读到内存中；
   3. 将压缩的图片数据解码成未压缩的位图形式，这是一个非常耗时的 CPU 操作；
   4. 最后 `Core Animation` 使用未压缩的位图数据渲染 `UIImageView` 的图层。

在上面的步骤中，我们提到了**图片的解压缩是一个非常耗时的 CPU 操作**，并且它**默认是在主线程**中执行的。那么当需要加载的图片比较多时，就会对我们应用的响应性造成严重的影响，尤其是在快速滑动的列表上，这个问题会表现得更加突出。这就是 UIImageView 的一个性能瓶颈。

> 实际上，当我们调用`[UIImage imageNamed:@"xxx"]`后，UIImage 中存储的是未解码的图片，而调用 `[UIImageView setImage:image]`后，会在主线程进行图片的解码工作并且将图片显示到 UI 上，这时候，UIImage 中存储的是解码后的 bitmap 数据。

### 为什么需要解压缩

既然图片的解压缩需要消耗大量的 CPU 时间，那么我们为什么还要对图片进行解压缩呢？是否可以不经过解压缩，而直接将图片显示到屏幕上呢？答案是否定的。要想弄明白这个问题，我们首先需要知道什么是位图。

**bitmap**：bitmap 又叫位图文件，它是一种非压缩的图片格式，所以体积非常大。所谓的非压缩，就是图片每个像素的原始信息在存储器中依次排列，一张典型的1920*1080像素的 bitmap 图片，每个像素由 RGBA 四个字节表示颜色，那么它的体积就是 1920 * 1080 * 4 = 1012.5kb。

由于 bitmap 简单顺序存储图片的像素信息，它可以不经过解码就直接被渲染到 UI 上。实际上，**其它格式的图片都需要先被首先解码为 bitmap，然后才能渲染到界面上**。

不管是 JPEG 还是 PNG 图片，都是一种压缩的位图图形格式。只不过 PNG 图片是无损压缩，并且支持 alpha 通道，而 JPEG 图片则是有损压缩，可以指定 0-100% 的压缩比。值得一提的是，在苹果的 SDK 中专门提供了两个函数用来生成 PNG 和 JPEG 图片：

```objc
// return image as PNG. May return nil if image has no CGImageRef or invalid bitmap format
UIKIT_EXTERN NSData * __nullable UIImagePNGRepresentation(UIImage * __nonnull image);

// return image as JPEG. May return nil if image has no CGImageRef or invalid bitmap format. compression is 0(most)..1(least)                           
UIKIT_EXTERN NSData * __nullable UIImageJPEGRepresentation(UIImage * __nonnull image, CGFloat compressionQuality);
```

**因此，在将磁盘中的图片渲染到屏幕之前，必须先要得到图片的原始像素数据，才能执行后续的绘制操作，这就是为什么需要对图片解压缩的原因。**

> 图片解压缩的过程其实就是将图片的二进制数据转换成像素数据的过程

> **图片的编码和解码**  
>  iOS 底层是用 ImageIO.framework 实现的图片编解码。目前 iOS 原生支持的格式有：JPEG、JPEG2000、PNG、GIF、BMP、ICO、TIFF、PICT，自 iOS 8.0 起，ImageIO 又加入了 APNG、SVG、RAW 格式的支持。在上层，开发者可以直接调用 ImageIO 对上面这些图片格式进行编码和解码。对于动图来说，开发者可以解码动画 GIF 和 APNG、可以编码动画 GIF。

**注意：图片所占内存的大小与图片的尺寸有关，而不是图片的文件大小**

## 色彩空间和像素格式

计算图片解码后每行需要的比特数，由两个参数相乘得到：每行的像素数 `width`，和存储一个像素需要的比特数4。

这里的4，其实是由每张图片的像素格式和像素组合来决定的，下表是苹果平台支持的像素组合方式。

![img](https:////upload-images.jianshu.io/upload_images/7271829-d82048891dbc0bb7)

表中的bpp，表示每个像素需要多少位；bpc表示颜色的每个分量，需要多少位。具体的解释方式，可以看下面这张图：

![img](https:////upload-images.jianshu.io/upload_images/7271829-b115f7ba11df4c42)

我们解码后的图片，默认采用 kCGImageAlphaNoneSkipLast RGB 的像素组合，没有 alpha 通道，每个像素32位4个字节，前三个字节代表红绿蓝三个通道, 但是有时候，如果我们只是绘制一个蒙版，是不需要这么字节表示的比如 Alpha 8 format，每个像素只需要占用 1 个字节，这之间的差距就造成了内存浪费。

### UIGraphicsImageRenderer 和 UIGraphicsBeginImageContextWithOptions

当我们为了离屏渲染，要创建 image buffer 时，我们通常会使用 UIGraphicsBeginImageContext，但是最好还是用 UIGraphicsImageRenderer，它的性能更好、更高效，并且支持广色域。这里有一个中间地带，如果你主要将图像渲染到图形图像渲染器(graphic image render)中，该图像可能使用超出 SRGB 色域的色彩空间值，但实际上并不需要更大的元素来存储这些信息。所以 UIImage 有一个可以用来获取预构建的 UIGraphicsImageRendererFormat 对象的 image renderer format 属性，该对象用于重新渲染图像时进行最优化存储。

所以苹果官方建议使用 UIGraphicsImageRenderer，这个方法是从 iOS 10 引入，在 iOS 12 上会自动选择最佳的图像格式，可以减少很多内存。系统可以根据图片分辨率选择创建解码图片的格式，如选用SRGB format 格式，每个像素占用 4 字节，而Alpha 8 format，每像素只占用 1 字节，可以减少大量的解码内存占用。

> 扩展阅读[色彩空间与像素格式](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.cnblogs.com%2Fleisure_chn%2Fp%2F10290575.html)

## imageWithContentsOfFile 和 imageNamed 对比

### imgeNamed

用这个方法加载图片分为两种情况:

1. 系统缓存有这个图片，直接从缓存中取得
2. 系统缓存没有这个图片
    通过传入的文件名对整个工程进行遍历 (在`application bundle`的顶层文件夹寻找名字的图象 ), 如果如果找到对应的图片,iOS 系统首先要做的是将这个图片放到系统缓存中去,以备下次使用的时候直接从系统缓存中取, 接下来重复第一步,即直接从缓存中取。
3. 由于系统会缓存图片，所以如果要加载的这个图片的文件量很多,文件大小很大,内存不足,内存泄露,甚至是程序的崩溃都是很容易发生的事.

### imageWithContentsOfFile

用这个方法只有一种情况,那就是仅仅加载图片, 图像数据不会被缓存. 因此在加载较大图片的时候, 以及图片使用情况很少的时候可以使用这两个方法 , 降低内存消耗.

> 加载本地图片，要比从Assets Catalogs耗时要长，具体见[Assets Catalogs 与 I/O 优化](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5cb74d786fb9a068773948fc)

## 图片内存优化

到此我们可知，图片经过解压之后，在内存中实际是根据图片的**分辨率**和图片渲染所用的**像素格式**而定的

### 一、对不常用的大图片，使用 `imageWithContentsOfFile` 代替 `imageNamed` 方法，避免内存缓存（相应的使用`imageNamed`要避免载入大量的图片造成内存暴增）

这是个老生常谈的问题了，我就简单说下，使用UIImage的imageNamed方法的时候，为了下次加快渲染速度，会缓存图片内存，所以对于使用不频繁的大图片进行缓存，非常耗费内存，可以使用imageWithContentsOfFile方法进行替换。

### 二、使用 ImageIO 方法，对大图片进行缩放，减少图片解码占用内存大小。(超大图片处理)

UIImage 在设置和调整大小的时候，需要将原始图像加压到内存中，然后对内部坐标空间做一系列转换，整个过程会消耗很多资源。我们可以使用 ImageIO，它可以直接读取图像大小和元数据信息，不会带来额外的内存开销。

这是( [WWDC2018 图像最佳实践](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5b1a7c2c5188257d5a30c820)）中推荐的方法，可以使用这个方法对图片进行缩放,UIImage 在设置和调整大小的时候，需要将原始图像加压到内存中，然后对内部坐标空间做一系列转换，整个过程会消耗很多资源。我们可以使用 ImageIO，它可以直接读取图像大小和元数据信息，不会带来额外的内存开销。
 这是官方的实例的Swift代码：

```swift
func downsample(imageAt imageURL: URL, to pointSize: CGSize, scale: CGFloat) -> UIImage {

    //生成CGImageSourceRef 时，不需要先解码。
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    let imageSource = CGImageSourceCreateWithURL(imageURL as CFURL, imageSourceOptions)!
    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    
    //kCGImageSourceShouldCacheImmediately 
    //在创建Thumbnail时直接解码，这样就把解码的时机控制在这个downsample的函数内
    let downsampleOptions = [kCGImageSourceCreateThumbnailFromImageAlways: true,
                                 kCGImageSourceShouldCacheImmediately: true,
                                 kCGImageSourceCreateThumbnailWithTransform: true,
                                 kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels] as CFDictionary
    //生成
    let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions)!
    return UIImage(cgImage: downsampledImage)
}
```

这是一种OC代码实现：

```objectivec
- (UIImage *)resizeScaleImage:(CGFloat)scale {
    
    CGSize imgSize = self.size;
    CGSize targetSize = CGSizeMake(imgSize.width * scale, imgSize.height * scale);
    NSData *imageData = UIImageJPEGRepresentation(self, 1.0);
    CFDataRef data = (__bridge CFDataRef)imageData;
    
    CFStringRef optionKeys[1];
    CFTypeRef optionValues[4];
    optionKeys[0] = kCGImageSourceShouldCache;
    optionValues[0] = (CFTypeRef)kCFBooleanFalse;
    CFDictionaryRef sourceOption = CFDictionaryCreate(kCFAllocatorDefault, (const void **)optionKeys, (const void **)optionValues, 1, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    CGImageSourceRef imageSource = CGImageSourceCreateWithData(data, sourceOption);
    CFRelease(sourceOption);
    if (!imageSource) {
        NSLog(@"imageSource is Null!");
        return nil;
    }
    //获取原图片属性
    int imageSize = (int)MAX(targetSize.height, targetSize.width);
    CFStringRef keys[5];
    CFTypeRef values[5];
    //创建缩略图等比缩放大小，会根据长宽值比较大的作为imageSize进行缩放
    keys[0] = kCGImageSourceThumbnailMaxPixelSize;
    CFNumberRef thumbnailSize = CFNumberCreate(NULL, kCFNumberIntType, &imageSize);
    values[0] = (CFTypeRef)thumbnailSize;
    keys[1] = kCGImageSourceCreateThumbnailFromImageAlways;
    values[1] = (CFTypeRef)kCFBooleanTrue;
    keys[2] = kCGImageSourceCreateThumbnailWithTransform;
    values[2] = (CFTypeRef)kCFBooleanTrue;
    keys[3] = kCGImageSourceCreateThumbnailFromImageIfAbsent;
    values[3] = (CFTypeRef)kCFBooleanTrue;
    keys[4] = kCGImageSourceShouldCacheImmediately;
    values[4] = (CFTypeRef)kCFBooleanTrue;
    
    CFDictionaryRef options = CFDictionaryCreate(kCFAllocatorDefault, (const void **)keys, (const void **)values, 4, &kCFTypeDictionaryKeyCallBacks, &kCFTypeDictionaryValueCallBacks);
    CGImageRef thumbnailImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, options);
    UIImage *resultImg = [UIImage imageWithCGImage:thumbnailImage];
    
    CFRelease(thumbnailSize);
    CFRelease(options);
    CFRelease(imageSource);
    CFRelease(thumbnailImage);
    
    return resultImg;
}
```

如果你还想学习更多的ImageIO的方法和参数使用，可以参考这篇文章[iOS中ImageIO框架详解与应用分析](https://links.jianshu.com/go?to=https%3A%2F%2Fmy.oschina.net%2Fu%2F2340880%2Fblog%2F838680)
如果你想自己测试一下内存占用效果，可以使用instruments 中的VMTracker进行测试，具体测试方法，可以看下这篇文章[iOS中的图片使用方式、内存对比和最佳实践](

### 三、绘制图片，用 UIGraphicsImageRenderer 代替 UIGraphicsBeginImageContextWithOptions，自动管理颜色格式

这个方法是[WWDC 2018：iOS 内存深入研究](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5b23dafee51d4558e03cbf4f%23heading-25)推荐的方法，
 使用 UIGraphicsBeginImageContextWithOptions 生成的图片，每个像素需要 4 个字节表示。建议使用 UIGraphicsImageRenderer，这个方法是从 iOS 10 引入，在 iOS 12 上会自动选择最佳的图像格式，可以减少很多内存。系统可以根据图片分辨率选择创建解码图片的格式，如选用SRGB format 格式，每个像素占用 4 字节，而Alpha 8 format，每像素只占用 1 字节，可以减少大量的解码内存占用。

## 解决UIImageView的性能瓶颈

### **我们在讨论UIImageView的性能瓶颈中发现，问题在于主线程进行图片解压缩占用了大量的CPU，解决问题的办法就是：在子线程提前对图片进行强制解压缩**

而强制解压缩的原理就是对图片进行重新绘制，得到一张新的解压缩后的位图。其中，用到的最核心的函数是 CGBitmapContextCreate 

```objc
/* Create a bitmap context. The context draws into a bitmap which is `width'
   pixels wide and `height' pixels high. The number of components for each
   pixel is specified by `space', which may also specify a destination color
   profile. The number of bits for each component of a pixel is specified by
   `bitsPerComponent'. The number of bytes per pixel is equal to
   `(bitsPerComponent * number of components + 7)/8'. Each row of the bitmap
   consists of `bytesPerRow' bytes, which must be at least `width * bytes
   per pixel' bytes; in addition, `bytesPerRow' must be an integer multiple
   of the number of bytes per pixel. `data', if non-NULL, points to a block
   of memory at least `bytesPerRow * height' bytes. If `data' is NULL, the
   data for context is allocated automatically and freed when the context is
   deallocated. `bitmapInfo' specifies whether the bitmap should contain an
   alpha channel and how it's to be generated, along with whether the
   components are floating-point or integer. */
CG_EXTERN CGContextRef __nullable CGBitmapContextCreate(void * __nullable data,
    size_t width, size_t height, size_t bitsPerComponent, size_t bytesPerRow,
    CGColorSpaceRef cg_nullable space, uint32_t bitmapInfo)
    CG_AVAILABLE_STARTING(__MAC_10_0, __IPHONE_2_0);
```

如果 `UIImage` 中存储的是已经解码后的数据，速度就会快很多，所以优化的思路就是：在子线程中对图片原始数据进行强制解码，再将解码后的图片抛回主线程继续使用，从而提高主线程的响应速度。
 我们需要使用的工具是 `Core Graphics` 框架的 `CGBitmapContextCreate` 方法和相关的绘制函数。总体的步骤是：

1. 创建一个指定大小和格式的 `bitmap context`。
2. 将未解码图片写入到这个 `context` 中，这个过程包含了强制解码。
3. 从这个 `context` 中创建新的 `UIImage` 对象，返回。

### SDWebImage 实现

下面是SDWebImage的核心代码：

```objective-c
// 1. 从 UIImage 对象中获取 CGImageRef 的引用。这两个结构是苹果在不同层级上对图片的表示方式，UIImage 属于 UIKit，是 UI 层级图片的抽象，用于图片的展示；CGImageRef 是 QuartzCore 中的一个结构体指针，用C语言编写，用来创建像素位图，可以通过操作存储的像素位来编辑图片。这两种结构可以方便的互转：
CGImageRef imageRef = image.CGImage;

// 2. 调用 UIImage 的 +colorSpaceForImageRef: 方法来获取原始图片的颜色空间参数。
CGColorSpaceRef colorspaceRef = [UIImage colorSpaceForImageRef:imageRef];
        
size_t width = CGImageGetWidth(imageRef);
size_t height = CGImageGetHeight(imageRef);

// 3. 计算图片解码后每行需要的比特数，由两个参数相乘得到：每行的像素数 width，和存储一个像素需要的比特数4(这里的4，其实是由每张图片的像素格式和像素组合来决定的)
size_t bytesPerRow = 4 * width;

// 4. 最关键的函数：调用 CGBitmapContextCreate() 方法，生成一个空白的图片绘制上下文，我们传入了上述的一些参数，指定了图片的大小、颜色空间、像素排列等等属性。
CGContextRef context = CGBitmapContextCreate(NULL,
                                             width,
                                             height,
                                             kBitsPerComponent,
                                             bytesPerRow,
                                             colorspaceRef,
                                             kCGBitmapByteOrderDefault|kCGImageAlphaNoneSkipLast);
if (context == NULL) {
    return image;
}
        
// 5. 调用 CGContextDrawImage() 方法，将未解码的 imageRef 指针内容，写入到我们创建的上下文中，这个步骤，完成了隐式的解码工作。
CGContextDrawImage(context, CGRectMake(0, 0, width, height), imageRef);

// 6. 从 context 上下文中创建一个新的 imageRef，这是解码后的图片了。
CGImageRef newImageRef = CGBitmapContextCreateImage(context);

// 7. 从 imageRef 生成供UI层使用的 UIImage 对象，同时指定图片的 scale 和 orientation 两个参数。
UIImage *newImage = [UIImage imageWithCGImage:newImageRef
                                        scale:image.scale
                                  orientation:image.imageOrientation];

CGContextRelease(context);
CGImageRelease(newImageRef);

return newImage;
```

通过以上的步骤，我们成功在子线程中对图片进行了强制转码，回调给主线程使用，从而大大提高了图片的渲染效率。这也是现在主流 App 和大量三方库的最佳实践。

### SDWebImage配置优化，减小CG-raster-data内存占用

在使用SDWebImage的时候，会默认保存图片解码后的内存，以便提高页面的渲染速度，但是这会导致内存的急速增加，所以可以在不影响体验的情况下，选择机型和系统，进行优化，避免大量的内存占用，引起OOM问题。关闭解码内存缓存的方法如下：

```objc
[[SDImageCache sharedImageCache] setShouldDecompressImages:NO];
[[SDWebImageDownloader sharedDownloader] setShouldDecompressImages:NO];
```

