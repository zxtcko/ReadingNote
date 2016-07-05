# ReadingNote
Daily reading material note
##一、[解读SDWebImage源码](http://ios.jobbole.com/86601/)

###重点
1.	创建一个组合Operation，是一个SDWebImageCombinedOperation对象，这个对象负责对下载operation创建和管理，同时有缓存功能，是对下载和缓存两个过程的组合。
2.	先去寻找这张图片 内存缓存和磁盘缓存，这两个功能在self.imageCache的queryDiskCacheForKey: done:方法中完成，这个方法的返回值既是一个缓存operation，最终被赋给上面的Operation的cacheOperation属性。在查找缓存的完成回调中的代码是重点：它会根据是否设置了SDWebImageRefreshCached选项和代理是否支持下载决定是否要进行下载，并对下载过程中遇到NSURLCache的情况做处理，还有下载失败的处理以及下载之后进行缓存，然后查看是否设置了形变选项并调用代理的形变方法进行对图片形变处理。
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