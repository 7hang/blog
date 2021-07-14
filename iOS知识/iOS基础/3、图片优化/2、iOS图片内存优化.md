## iOS图片内存优化

### 前言

对于 iOS 系统而言，绝大部分场景下哪类数据占内存最多呢？当然是图片！需要注意的是，图片所占内存的大小与图片的尺寸有关，而不是图片的文件大小。
 例如：有一个 590KB 的图片，分辨率是 2048px * 1536px，它实际使用的内存不是 590KB，而是2048 * 1536 * 4 = 12 MB。

**至于为什么图片占用这么大的内存，而不是图片的原始大小？**

这就要从图片格式来说，我们通常用的图片格式如：png和jpeg等，这些格式的图片都是压缩的位图格式，不能直接渲染展示在屏幕上，所以就需要在渲染到屏幕之前，需要将图片解压缩，得到图片的原始像素数据，过程如下：

```php
Data buffer(图片的元数据) ->Image buffer(图片解码后在内存中的表示) ->Frame buffer(代表了一帧在内存中的表示) 
```

### 优化方法

#### 一、对不常用的大图片，使用imageWithContentsOfFile代替imageNamed方法，避免内存缓存。

这是个老生常谈的问题了，我就简单说下，使用UIImage的imageNamed方法的时候，为了下次加快渲染速度，会缓存图片内存，所以对于使用不频繁的大图片进行缓存，非常耗费内存，可以使用imageWithContentsOfFile方法进行替换。

#### 二、使用ImageIO方法，对大图片进行缩放，减少图片解码占用内存大小。

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
如果你想自己测试一下内存占用效果，可以使用instruments 中的VMTracker进行测试，具体测试方法，可以看下这篇文章[iOS中的图片使用方式、内存对比和最佳实践](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5b2ddfa7e51d4553156be305)

#### 三、用 UIGraphicsImageRenderer 代替 UIGraphicsBeginImageContextWithOptions，

这个方法是[WWDC 2018：iOS 内存深入研究](https://links.jianshu.com/go?to=https%3A%2F%2Fjuejin.im%2Fpost%2F5b23dafee51d4558e03cbf4f%23heading-25)推荐的方法，
 使用 UIGraphicsBeginImageContextWithOptions 生成的图片，每个像素需要 4 个字节表示。建议使用 UIGraphicsImageRenderer，这个方法是从 iOS 10 引入，在 iOS 12 上会自动选择最佳的图像格式，可以减少很多内存。系统可以根据图片分辨率选择创建解码图片的格式，如选用SRGB format 格式，每个像素占用 4 字节，而Alpha 8 format，每像素只占用 1 字节，可以减少大量的解码内存占用。

#### 四、SDWebImage配置优化，减小CG-raster-data内存占用

在使用SDWebImage的时候，会默认保存图片解码后的内存，以便提高页面的渲染速度，但是这会导致内存的急速增加，所以可以在不影响体验的情况下，选择机型和系统，进行优化，避免大量的内存占用，引起OOM问题。关闭解码内存缓存的方法如下：

```css
[[SDImageCache sharedImageCache] setShouldDecompressImages:NO];
[[SDWebImageDownloader sharedDownloader] setShouldDecompressImages:NO];
```