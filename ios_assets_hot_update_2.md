# 实现iOS图片等资源文件的热更新化(二):自定义的动态 imageNamed

这篇文章,要解决的是,使用一个自定义的 imageNamed 函数来替代系统的 imageNamed 函数.内部逻辑,将贯穿对比论证 关于"合适"的图片的定义.对iOS加载图片的规则不是很熟悉的童鞋,可以着重看这篇.

## 不同后缀图片加载的优先级

* iPhone 7 plus(iOS10.0): sample@3x.png > sample@2x.png > sample~iphone.png >sample.png
其他后缀的图片总是不被加载.

* iPad Pro 12.9 inch(iOS10.0): sample@2x.png > sample~ipad.png > sample.png 其他后缀的图片总是不被加载.


不同后缀的图片|iPhone 7 plus(iOS10.0)|iPad Pro 12.9 inch(iOS10.0)
-|-|-
sample.png|7|8
sample@2x.png|9|10
sample@3x.png|10|0
sample~iphone.png|8|0
sample~iphone@2x.png|0|0
sample~iphone@3x.png|0|0
sample~ipad.png|0|9
sample~ipad@2x.png|0|0

可以使用同名不同内容的图片来对比观察.优先级从高到低.优先级较高的优先被加载,优先级为0的永远不会被加载.仅以iPhone 7 plus 和 iPad Pro为例分析,其他情况可自行.所用验证版本为iOS10,未来不同机型手机和系统可能会有差异.

想自己动手试一下的,可以下载示例: [https://github.com/ios122/ios_assets_hot_update/raw/master/res/ios_assets_hot_update_2.zip](https://github.com/ios122/ios_assets_hot_update/raw/master/res/ios_assets_hot_update_2.zip) 很小,只有100多K.编译,我此时用的是 Xcode 8.

## 使用bundle包放置图片等资源文件

![使用bundle包放置图片等资源文件](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_2/001.png?raw=true)

资源把到一个bundle包中,便于保留资源的目录结构,也方便整体管理与替换.iOS中的bundle包,就一个一个特殊的以.bunle结尾的文件夹.示例中,我使用的是main.bundle.另外,关于bundle保留资源目录结构这个特点,是react-native中很依赖的一个特性,以后你的项目中或许也会需要.如果单单只是从原有 Images.xcassets 迁移代码的话,此处都放于同一层级即可.

## 使用 **imageWithContentsOfFile:** 加载图片

把图片放到资源文件夹main.bundle后,再加载图片,可以参考下面的代码,这样做的额外的好处就是可以适当减小图片加载的内存占用问题:

```oc
NSString * bundlePath = [[NSBundle mainBundle].resourcePath stringByAppendingPathComponent:@"main.bundle"];
NSBundle * mainBundle = [NSBundle bundleWithPath:bundlePath];
NSString * imgPath = [mainBundle pathForResource:@"sample" ofType:@"png"];
self.sampleImageView.image = [UIImage imageWithContentsOfFile: imgPath];
```

## 使iPhone @3x 与iPad @2x 共用同一张图片

首先是需要显示加载 @3x 的图片:

```oc
NSString * imgPath = [mainBundle pathForResource:@"sample@3x" ofType:@"png"];
```

此时代码,在iPhone 7 / iPhone 7 plus/ iPad Pro,都能显示图片,直接输出图片本身的尺寸都为 原图尺寸的 1/3.

```
NSLog(@"加载后的图片尺寸:%@",[NSValue valueWithCGSize:self.sampleImageView.image.size]);
```

但是,此处有一个问题.@3x总是被解读为三倍图,在iPhone上,正是我们需要的尺寸,但是在iPad上,尺寸就有些偏小了.我们在iPad上,通常总是需要将此张图按照@2x图来显示.这是一个规律!做过iPhone和iPad通用图标尺寸适配的童鞋,应该早就注意到了.

所以,现在要解决的关键技术问题是:如何把 @3x图,在iPad上按照@2x图来解读?相对完整代码如下,最终输出的图片尺寸在iPhone上为原始尺寸的1/3,在iPad上为原始尺寸的1/2,正是我们需要的:

```oc
 NSString * bundlePath = [[NSBundle mainBundle].resourcePath stringByAppendingPathComponent:@"main.bundle"];
 NSBundle * mainBundle = [NSBundle bundleWithPath:bundlePath];
 NSString * imgPath = [mainBundle pathForResource:@"sample@3x" ofType:@"png"];

 UIImage * image;
 static NSString * model;

 if (!model) {
     model = [[UIDevice currentDevice]model];
 }

 if ([model isEqualToString:@"iPad"]) {
     NSData *imageData = [NSData dataWithContentsOfFile: imgPath];
     image = [UIImage imageWithData:imageData scale:2.0];
 }else{
     image = [UIImage imageWithContentsOfFile: imgPath];
 }

 self.sampleImageView.image = image;

 NSLog(@"加载后的图片尺寸:%@",[NSValue valueWithCGSize:self.sampleImageView.image.size]);
```

## 封装为类目(category),实现自定义的 imageNamed

此处实现了一个简单够用的类目方法,支持从指定bundle读取指定名字的图片:

```oc
#import "UIImage+imageNamed_bundle_.h"

@implementation UIImage (imageNamed_bundle_)
+ (UIImage *)imageNamed:(NSString *)imgName bundle:(NSString *)bundleName
{
    bundleName = [NSString stringWithFormat:@"%@.bundle",bundleName];
    imgName = [NSString stringWithFormat:@"%@@3x",imgName];

    NSString * bundlePath = [[NSBundle mainBundle].resourcePath stringByAppendingPathComponent: bundleName];
    NSBundle * mainBundle = [NSBundle bundleWithPath:bundlePath];
    NSString * imgPath = [mainBundle pathForResource:imgName ofType:@"png"];

    UIImage * image;
    static NSString * model;

    if (!model) {
        model = [[UIDevice currentDevice]model];
    }

    if ([model isEqualToString:@"iPad"]) {
        NSData *imageData = [NSData dataWithContentsOfFile: imgPath];
        image = [UIImage imageWithData:imageData scale:2.0];
    }else{
        image = [UIImage imageWithContentsOfFile: imgPath];
    }
    return  image;
}
@end
```

借助类目,原来的调用,可以简化为:

```oc
UIImage * image = [UIImage imageNamed:@"sample" bundle:@"main"];
self.sampleImageView.image = image;
```

也支持有层级结构的图片资源的读取呦:

```oc
UIImage * image = [UIImage imageNamed:@"sub/sample" bundle:@"main"];
self.sampleImageView.image = image;
```

## 参考链接

* [http://stackoverflow.com/questions/4976005/image-from-url-for-retina-display](http://stackoverflow.com/questions/4976005/image-from-url-for-retina-display)

* [http://stackoverflow.com/questions/11197509/ios-how-to-get-device-make-and-model](http://stackoverflow.com/questions/11197509/ios-how-to-get-device-make-and-model)

## 系列专属 github 项目主页

[https://github.com/ios122/ios_assets_hot_update](https://github.com/ios122/ios_assets_hot_update)
