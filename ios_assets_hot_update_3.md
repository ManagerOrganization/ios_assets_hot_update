# 实现iOS图片等资源文件的热更新化(三):动态的资源文件夹

## 简介

此文,将尝试动态从某个不确定的文件夹中加载资源文件.文章,会继续完善自定义的 imageNamed 函数,并为下一篇文章铺垫.

## 这么做的意义

正如我们经常所说的那样,大多数情景知道做事的意义往往比做事的方法本身更有意义.意义本身,往往蕴含着目的,最终的需求一类的东西;而方法,只是我们暂时寻找的用来达到最终的目的采取的一种可行的手段.知晓意义本身的意义在于,在以后的以后,我们有可能找到更合适的方法来实现目的;也就是我们所说的,到知识的丰富性得到一定程度之后,许多人在自己的个人技能提升过程中,多少总会有那种融会贯通,一通百通的情况出现.可以肯定的是,那种醍醐灌顶的感觉,肯定不是单纯的编码行数的变化的引起的;更多的,是由于你在有意无意中关于某个编码需求本身的意义的探寻所促成的.

具体到这里,我们为什么需要动态的资源文件夹呢?就目前的探讨本身所透露出来的信息而言,主要是因为我们的main.bundle放在了app里,而iOS App本身的打包进去的文件,在用户手机上是只读的.这样表述,有三层含义:

1. 如果你的资源文件是放置在App ipa包里的,尝试直接更新它,是不可能的 -- 至少对于一个native的 iOS App 是这样;
2. 如果你的main.bundle是从网上动态下载的,每次下载都放置到用户文件夹特定位置,那你的确是不需要考虑过多动态资源文件夹的;
3. 如果某一天iOS机制的发生变化,或者你为其他平台编写app,但是其本身的App资源文件是可写的,那你也很可能是可以不用动态资源文件夹的;

## 从特定的缓存目录读取资源文件

从特定的缓存目录读取加载资源文件,可以看做动态资源文件夹的一种特殊形式,所以我们先试着处理这种单一的情况.

### 1.动态拼接处特定的缓存目录

在iOS App中, **固定** 的缓存目录和 **特定** 的缓存目录,还是有区别的.主要是因为真机上iOS App每次启动时,其对应的文件目录是动态变化的.也就是说,我们以后如果有存储文件路径的需求,一定要记住只能存储文件相对于程序沙盒主目录 **NSHomeDirectory** 的相对路径.顺便说一句,主目录的程序主目录的可见子目录有3个,分别是: *Documents* , *Library* , *tmp* ,具体介绍,可参考博文: [ iOS沙盒文件读写](http://blog.csdn.net/yangbingbinga/article/details/46406053)

>* Documents：苹果建议将程序创建产生的文件以及应用浏览产生的文件数据保存在该目录下，iTunes备份和恢复的时候会包括此目录  
* Library：存储程序的默认设置或其它状态信息；  
* Library/Caches：存放缓存文件，保存应用的持久化数据，用于应用升级或者应用关闭后的数据保存，不会被itunes同步，所以为了减少同步的时间，可以考虑将一些比较大的文件而又不需要备份的文件放到这个目录下。  
* tmp：提供一个即时创建临时文件的地方，但不需要持久化，在应用关闭后，该目录下的数据将删除，也可能系统在程序不运行的时候清除。

现在我们的资源目录,将假定固定放在相对目录 *Library/Caches/patch* 中,其名为 *main.bundle*

那么在需要时,我们就可以这样访问到我们的资源文件夹:

```objc
NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
NSString * cacheBundleDir = [[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches/Patch/"];
NSLog(@"缓存资源目录: %@", cacheBundleDir); // 模拟器示例输出: 缓存资源目录: /Users/yanfeng/Library/Developer/CoreSimulator/Devices/930159D0-850F-43CE-88D2-08BE9D4A7E7F/data/Containers/Data/Application/EE3A92AB-2DBE-44C5-9103-11BAC7AECE15/Library/Caches/Patch/
```

### 2.App第一次初始化时,将资源文件复制到特定缓存目录

```objc
NSString * bundleName = @"main.bundle";
NSError * err = nil;
NSFileManager * defaultManager = [NSFileManager defaultManager];
if ( ! [defaultManager fileExistsAtPath:cacheBundleDir]) {
    [defaultManager createDirectoryAtPath:cacheBundleDir withIntermediateDirectories:YES attributes:nil error: &err];

    if(err){
        NSLog(@"初始化目录出错:%@", err);
    }

    NSString * defaultBundlePath = [[NSBundle mainBundle].resourcePath stringByAppendingPathComponent: bundleName];

    NSString * cacheBundlePath = [cacheBundleDir stringByAppendingPathComponent:bundleName];
    [defaultManager copyItemAtPath: defaultBundlePath toPath:cacheBundlePath error: &err];

    if(err){
        NSLog(@"复制初始资源文件出错:%@", err);
    }
}
```

代码,基本就像上面那样,有几个点我想着重说一下:

1. *fileExistsAtPath* 判定 缓存目录的有无来判定是否是第一次启动.这个逻辑,在真实的补丁逻辑中,很可能是不严密的,后续会使用其他方式,此处够用即可;
2. *createDirectoryAtPath* 用于目录不存在时,先构建目录的层级结构;否则如果直接复制,很有可能会报错的 -- 这取决于你的复制的目标目录与已有目录的层级差是否为1;
3. *copyItemAtPath:toPath:* 的 *toPath* 是一个完整的且不存在的目标路径,不一定非得与 *copyItemAtPath* 参数的最后一级路径同名,此处仅为简化处理;以后如果有需要,此函数是可以通过同时执行复制和重命名两个操作的,如将 *main.bundle* 重名为 *default.bundle* ;
4. 代码最好放在 *AppDelegate.m* 中;
5. 在模拟器上,你可以很容易地看到函数执行后的效果:右击finder --> 前往文件夹 --> 输入Xcode输出的 *缓存资源目录*.

![前往文件夹](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_3/01.png?raw=true)
![输入Xcode输出的 *缓存资源目录*](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_3/02.png?raw=true)
![模拟器结果](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_3/03.png?raw=true)

### 3.从特定缓存目录加载文件

因为目录是特定的,我们只要每次App启动后,根据相对路径动态获取绝对路径,进而拿到 缓存目录中 *main.bundle* 资源包路径,然后就可以使用已有的方法,从 *bundle* 里取图片即可:

```objc
NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
NSString * cacheBundleDir = [[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches/Patch/"];

NSString * bundleName = @"main.bundle";
NSString * imgName = @"sample@3x";

NSString * bundlePath = [cacheBundleDir stringByAppendingPathComponent: bundleName];
NSBundle * cacheMainBundle = [NSBundle bundleWithPath:bundlePath];
NSString * imgPath = [cacheMainBundle pathForResource:imgName ofType:@"png"];
UIImage * image = [UIImage imageWithContentsOfFile: imgPath];
self.sampleImageView.image = image;
```

## 从动态的缓存目录读取资源文件

这里,主要是和[实现iOS图片等资源文件的热更新化(二):自定义的动态 imageNamed](http://www.ios122.com/2016/09/ios_assets_hot_update_2/)的类目方法结合扩展下,使原来的类目扩展支持从动态的缓存目录读取bundle,思路本身也很简单,只要更改下用于确定bundle位置处的代码即可:

```objc
+ (UIImage *)imageNamed:(NSString *)imgName bundle:(NSString *)bundleName cacheDir:(NSString *)cacheDir
{
    NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);

    NSString * cacheBundleDir = [NSBundle mainBundle].resourcePath;

    if (cacheDir) {
        cacheBundleDir = [[[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches"] stringByAppendingPathComponent:cacheDir];
    }

    bundleName = [NSString stringWithFormat:@"%@.bundle",bundleName];
    imgName = [NSString stringWithFormat:@"%@@3x",imgName];

    NSString * bundlePath = [cacheBundleDir stringByAppendingPathComponent: bundleName];
    NSBundle * mainBundle = [NSBundle bundleWithPath:bundlePath];
    NSString * imgPath = [mainBundle pathForResource:imgName ofType:@"png"];

    UIImage * image;
    static NSString * model;

    if (!model) {
        model = [[UIDevice currentDevice]model];
    }

    if ([model isEqualToString:@"iPad"]) {
        NSData * imageData = [NSData dataWithContentsOfFile: imgPath];
        image = [UIImage imageWithData:imageData scale:2.0];
    }else{
        image = [UIImage imageWithContentsOfFile: imgPath];
    }
    return  image;
}
```

原来的从 ipa 包中加载 资源文件的逻辑可以基于此方法重写:

```objc
+ (UIImage *)imageNamed:(NSString *)imgName bundle:(NSString *)bundleName
{
    return [self imageNamed:imgName bundle:bundleName cacheDir:nil];
}
```
*+ (UIImage *)imageNamed:(NSString *)imgName bundle:(NSString *)bundleName cacheDir:(NSString *)cacheDir* 方法中的 *cacheDir* 也是支持多级目录的,如:

```objc
UIImage * image = [UIImage imageNamed:@"sub/sample" bundle:@"main" cacheDir:@"patch/default"];
self.sampleImageView.image = image;
```

注意,此时初始复制到缓存目录的逻辑,也是适当对应子目录变更下:

```objc
NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);
NSString * cacheBundleDir = [[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches/Patch/default/"];
NSLog(@"缓存资源目录: %@", cacheBundleDir);
```

## 参考资源:

* [ iOS沙盒文件读写](http://blog.csdn.net/yangbingbinga/article/details/46406053)
* 
[本节内容完整可执行Xcode工程代码,只有100k](https://github.com/ios122/ios_assets_hot_update/raw/master/res/ios_assets_hot_update_3.zip)
* [系列文章,专属github项目](https://github.com/ios122/ios_assets_hot_update)
