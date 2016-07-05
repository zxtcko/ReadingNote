# ReadingNote
Daily reading material note
##一、[解读SDWebImage源码](http://ios.jobbole.com/86601/)

###重点
1.	创建一个组合Operation，是一个SDWebImageCombinedOperation对象，这个对象负责对下载operation创建和管理，同时有缓存功能，是对下载和缓存两个过程的组合。
2.	根据图片的url生成key，根据key寻找 内存缓存和磁盘缓存，这两个功能在self.imageCache的queryDiskCacheForKey: done:方法中完成，这个方法的返回值既是一个缓存operation，最终被赋给上面的Operation的cacheOperation属性。在查找缓存的完成回调中的代码是重点：它会根据是否设置了SDWebImageRefreshCached选项和代理是否支持下载决定是否要进行下载，并对下载过程中遇到NSURLCache的情况做处理，还有下载失败的处理以及下载之后进行缓存，然后查看是否设置了形变选项并调用代理的形变方法进行对图片形变处理。
3.	将上面的下载方法返回的操作命名为subOperation，并在组合操作operation的cancelBlock代码块中添加对subOperation的cancel方法的调用。

```Objective-C

// UIView+WebCacheOperation
- (void)sd_setImageWithURL:(NSURL *)url
          placeholderImage:(UIImage *)placeholder
                   options:(SDWebImageOptions)options
                  progress:(SDWebImageDownloaderProgressBlock)progressBlock 
completed:(SDWebImageCompletionBlock)completedBlock {
    [self sd_cancelCurrentImageLoad];
    //将 url作为属性绑定到ImageView上,用static char imageURLKey作key
    objc_setAssociatedObject(self, &imageURLKey, url, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
 
/*options & SDWebImageDelayPlaceholder这是一个位运算的与操作,!(options & SDWebImageDelayPlaceholder)的意思就是options参数不是SDWebImageDelayPlaceholder,就执行以下操作   
 
#define dispatch_main_async_safe(block)\
    if ([NSThread isMainThread]) {\
        block();\
    } else {\
        dispatch_async(dispatch_get_main_queue(), block);\
    }
*/
这是一个宏定义,因为图像的绘制只能在主线程完成,所以dispatch_main_sync_safe就是为了保证block在主线程中执行
 if (!(options & SDWebImageDelayPlaceholder)) {
        dispatch_main_async_safe(^{
//设置imageView的placeHolder
            self.image = placeholder;
        });
    }
 
    if (url) {
 
        // 检查是否通过`setShowActivityIndicatorView:`方法设置了显示正在加载指示器。如果设置了，使用`addActivityIndicator`方法向self添加指示器
        if ([self showActivityIndicatorView]) {
            [self addActivityIndicator];
        }
 
        __weak __typeof(self)wself = self;
//下载的核心方法
        id  operation = [SDWebImageManager.sharedManager downloadImageWithURL:url options:options 
progress:progressBlock completed:^(UIImage *image, NSError *error, SDImageCacheType cacheType, BOOL finished, NSURL *imageURL) {
          //移除加载指示器
            [wself removeActivityIndicator];
          //如果imageView不存在了就return停止操作
            if (!wself) return;
            dispatch_main_sync_safe(^{
                if (!wself) return;
/*
SDWebImageAvoidAutoSetImage,默认情况下图片会在下载完毕后自动添加给imageView,但是有些时候我们想在设置图片之前加一些图片的处理,就要下载成功后去手动设置图片了,不会执行`wself.image = image;`,而是直接执行完成回调，有用户自己决定如何处理。
*/
                if (image && (options & SDWebImageAvoidAutoSetImage) && completedBlock)
                {
                    completedBlock(image, error, cacheType, url);
                    return;
                }
/*
如果后两个条件中至少有一个不满足，那么就直接将image赋给当前的imageView
，并调用setNeedsLayout
*/
                else if (image) {
                    wself.image = image;
                    [wself setNeedsLayout];
                } else {
/*
image为空，并且设置了延迟设置占位图，会将占位图设置为最终的image，，并将其标记为需要重新布局。
  */
                    if ((options & SDWebImageDelayPlaceholder)) {
                        wself.image = placeholder;
                        [wself setNeedsLayout];
                    }
                }
                if (completedBlock && finished) {
                    completedBlock(image, error, cacheType, url);
                }
            });
        }];
 // 为UIImageView绑定新的操作,以为之前把ImageView的操作cancel了
        [self sd_setImageLoadOperation:operation forKey:@"UIImageViewImageLoad"];
    } else {
 // 判断url不存在，移除加载指示器，执行完成回调，传递错误信息。
        dispatch_main_async_safe(^{
            [self removeActivityIndicator];
            if (completedBlock) {
                NSError *error = [NSError errorWithDomain:SDWebImageErrorDomain code:-1 userInfo:@{NSLocalizedDescriptionKey : @"Trying to load a nil url"}];
                completedBlock(nil, error, SDImageCacheTypeNone, url);
            }
        });
    }
}
```


*	SDWebImageManager

```Objective-C
/**
 * The SDWebImageManager is the class behind the UIImageView+WebCache category and likes.
 * It ties the asynchronous downloader (SDWebImageDownloader) with the image cache store (SDImageCache).
 * You can use this class directly to benefit from web image downloading with caching in another context than
 * a UIView.
 *
 * Here is a simple example of how to use SDWebImageManager:
 *
 * <a href='http://www.jobbole.com/members/java12'>@code</a>
/*
概述了SDWenImageManager的作用,其实UIImageVIew+WebCache这个Category背后执行操作的就是这个SDWebImageManager.它会绑定一个下载器也就是SDwebImageDownloader和一个缓存SDImageCache
*/
 
 
/**
 * Downloads the image at the given URL if not present in cache or return the cached version otherwise.
   若图片不在cache中,就根据给定的URL下载图片,否则返回cache中的图片 
 *
 * @param url            The URL to the image
 * @param options        A mask to specify options to use for this request
 * @param progressBlock  A block called while image is downloading
 * @param completedBlock A block called when operation has been completed. 
 *
 *   This parameter is required. 
 * 
 *   This block has no return value and takes the requested UIImage as first parameter.
 *   In case of error the image parameter is nil and the second parameter may contain an NSError.
 *
 *   The third parameter is an `SDImageCacheType` enum indicating if the image was retrieved from the local cache
 *   or from the memory cache or from the network.
 *
 *   The last parameter is set to NO when the SDWebImageProgressiveDownload option is used and the image is 
 *   downloading. This block is thus called repeatedly with a partial image. When image is fully downloaded, the
 *   block is called a last time with the full image and the last parameter set to YES.
 *
 * @return Returns an NSObject conforming to SDWebImageOperation. Should be an instance of SDWebImageDownloaderOperation
 */
 
/*
*第一个参数是必须的,就是image的url
*第二个参数options你可以定制化各种各样的操作
*第三个参数是一个回调block,用于图片下载过程中的回调
*第四个参数是一个下载完成的回调,会在图片下载完成后回调
*返回值是一个NSObject类,并且这个NSObject类是遵循一个协议这个协议叫SDWebImageOperation,这个协议里面只写了一个协议,就是一个cancel一个operation的协议
*/
- (id )downloadImageWithURL:(NSURL *)url
                                         options:(SDWebImageOptions)options
                                        progress:(SDWebImageDownloaderProgressBlock)progressBlock
                                       completed:(SDWebImageCompletionWithFinishedBlock)completedBlock;
```

*	检查一个图片是否被缓存的方法：
	1.	先判断是否在内存中，如果存在，主线程回调block，如果不存在，查询是否在磁盘中
	2.	查询是否在磁盘中，主线程回调block

```Objective-C

- (BOOL)cachedImageExistsForURL:(NSURL *)url {
  //调用上面的方法取到image的url对应的key
    NSString *key = [self cacheKeyForURL:url];
//首先检测内存缓存中时候存在这张图片,如果已有直接返回yes 
    if ([self.imageCache imageFromMemoryCacheForKey:key] != nil) return YES;
//如果内存缓存里面没有这张图片,那么就调用diskImageExistsWithKey这个方法去硬盘找
    return [self.imageCache diskImageExistsWithKey:key];
}
 
// 检测硬盘里是否缓存了图片
- (BOOL)diskImageExistsForURL:(NSURL *)url {
 NSString *key = [self cacheKeyForURL:url];
 return [self.imageCache diskImageExistsWithKey:key];
}
```

*	在搜索一个个元素的时候NSSet比NSArray效率高,主要是它用到了一个算法hash(散列,哈希) ,比如你要存储A,一个hash算法直接就能找到A应该存储的位置;同样当你要访问A的时候,一个hash过程就能找到A存储的位置,对于NSArray,若想知道A到底在不在数组中,则需要遍历整个数据,显然效率较低了