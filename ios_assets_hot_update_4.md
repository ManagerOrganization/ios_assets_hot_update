# 实现iOS图片等资源文件的热更新化(四): 一个最小化的补丁更新逻辑

## 简介

![逻辑图](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_4/0.png?raw=true)

以前写过一个补丁更新的文章,此处会做一个更精简的最小化实现,以便于集成.为了使逻辑具有通用性,将剥离对AFNetworking和ReativeCocoa的依赖.原来的文章,可以先看这里: [http://www.ios122.com/2015/12/jspatconline/](http://www.ios122.com/2015/12/jspatconline/)


## 这么做的意义

先交代动机和意义,或许应该成为自己博客的一个标准框架内容之一,不然以后自己需要看着,也不过是一堆干瘪的代码.基本的逻辑图,如上!此处,我就从简!

从简的原因有3:
1. 补丁更新,状态可以设计的很复杂,就像开头那篇文章提到的那样,但是我感觉没多大必要,至少在我们的App中;
2. 我想演示一个相对完整的逻辑,但是又不想耗费太多的时间构建场景;
3. 从简后的方案,简单但够用了,至少目前针对我们的项目来说;

所以说:这篇文章的意义,其实是在于简化已有的热更新代码,越简单越好维护.

## 基本思路

1. App启动时,判断特定的服务器接口所返回的图片url是否为最新,判断方式就是比对返回值中的md5字段与本地保存的资源的url是否一致;
2. 如果图片资源有更新,则下载解压到指定的缓存目录,初步打算以资源文件的md5来划分文件夹,来避免冲突;
3. 读取图片时,优先从缓存目录读取,缓存目录不存在再从ipa资源包中读取;

下面就一步一步来实现了.

## App启动时,判断有无最新图片资源

此处主要涉及到的可能的技术点:

### 1. 如何用基础的网络类库发送网络请求?

先简单封装一个函数来获取,用到了block.block经常用,但到现在都记不太清形式,大都是从其他处copy下,然后改改参数.记不住,也懒得记!

```objc
- (void)fetchPatchInfo:(NSString *) urlStr completionHandler:(void (^)(NSDictionary * patchInfo, NSError * error))completionHandler
{
    NSURLSessionConfiguration * defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession * defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate: self delegateQueue: [NSOperationQueue mainQueue]];

    NSURL * url = [NSURL URLWithString:urlStr];

    NSURLSessionDataTask * dataTask = [defaultSession dataTaskWithURL:url
                                                    completionHandler:^(NSData * data, NSURLResponse * response, NSError * error) {
                                                        NSDictionary * patchInfo = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];;

                                                        completionHandler(patchInfo, error);
                                                    }];

    [dataTask resume];
}
```

基于block,调用的代码也就很简答了.

```objc
[self fetchPatchInfo: @"https://raw.githubusercontent.com/ios122/ios_assets_hot_update/master/res/patch_04.json"
 completionHandler:^(NSDictionary * patchInfo, NSError * error) {
     if ( ! error) {
         NSLog(@"patchInfo:%@", patchInfo);
     }else
     {
         NSLog(@"fetchPatchInfo error: %@", error);
     }
 }];
```

好吧,我承认AFNetworking用习惯了,好久没用原始的网络请求的代码了,有点low,莫怪!

### 2. 如何校验下载的文件的md5值,如果你需要的话?

开头那篇文章链接里,有提到.核心,其实是在于下载文件之后,md5值的计算,剩余的就是字符串比较操作了.

注意要先引入系统库

```objc
 #include <CommonCrypto/CommonDigest.h>
```

```objc
/**
 *  获取文件的md5信息.
 *
 *  @param path 文件路径.
 *
 *  @return 文件的md5值.
 */
-(NSString *)mcMd5HashOfPath:(NSString *)path
{
    NSFileManager * fileManager = [NSFileManager defaultManager];

    // 确保文件存在.
    if( [fileManager fileExistsAtPath:path isDirectory:nil] )
    {
        NSData * data = [NSData dataWithContentsOfFile:path];
        unsigned char digest[CC_MD5_DIGEST_LENGTH];
        CC_MD5( data.bytes, (CC_LONG)data.length, digest );

        NSMutableString * output = [NSMutableString stringWithCapacity:CC_MD5_DIGEST_LENGTH * 2];

        for( int i = 0; i < CC_MD5_DIGEST_LENGTH; i++ )
        {
            [output appendFormat:@"%02x", digest[i]];
        }

        return output;
    }
    else
    {
        return @"";
    }
}
```

### 3. 使用什么保存与获取本地缓存资源的md5等信息?

好吧,我打算直接使用用户配置文件,

```objc
NSString * source_patch_key = @"SOURCE_PATCH";

[[NSUserDefaults standardUserDefaults] setObject:patchInfo forKey: source_patch_key];
patchInfo = [[NSUserDefaults standardUserDefaults] objectForKey: source_patch_key];

NSLog(@"patchInfo:%@", patchInfo);
```

## 补丁下载与解压

此处主要涉及到的可能的技术点:

### 1. 如何基于图片缓存信息来找到指定的缓存目录?

问题本身有些绕口,其实我想做的就是根据补丁的md5,放到不同的缓存文件夹,如补丁md5为 e963ed645c50a004697530fa596f180b,则对应放到 patch/e963ed645c50a004697530fa596f180b 文件夹.封装一个简单的根据md5返回缓存路径的方法吧:

```objc
- (NSString *)cachePathFor:(NSString * )patchMd5
{
    NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);

    NSString * cachePath = [[[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches/patch"] stringByAppendingPathComponent:patchMd5];

    return cachePath;
}
```

使用时,类似这样:

```objc
NSString * urlStr = [patchInfo objectForKey: @"url"];

[weak_self downloadFileFrom:urlStr completionHandler:^(NSURL * location, NSError * error) {
    if (error) {
        NSLog(@"download file url:%@  error: %@", urlStr, error);
        return;
    }

    NSString * cachePath = [weak_self cachePathFor: [patchInfo objectForKey:@"md5"]];
    NSLog(@"location:%@ cachePath:%@",location, cachePath);

}];
```

### 2. 如何解压文件到指定目录?

![在模拟中查看解压后的文件](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_4/1.png?raw=true)

如果需要安装 CocoaPods ,建议使用 `brew`:

```shell
brew install CocoaPods
```

解压本身推荐 *SSZipArchive* 库,一行代码搞定:

```objc
[SSZipArchive unzipFileAtPath:location.path toDestination: patchCachePath overwrite:YES password:nil error:&error];
```

### 3. 在什么时候更新本地的缓存资源的相关信息?

建议是在下载并解压资源文件到指定缓存目录后,再更新补丁的相关缓存信息,因为这个信息,读取图片时,也是需要的.如果先删除某个补丁,按照目前的设计,一种比较偷懒的方案就是,在服务器上放上一个新的空资源文件就可以了.

```objc
NSString * source_patch_key = @"SOURCE_PATCH";

[[NSUserDefaults standardUserDefaults] setObject:patchInfo forKey: source_patch_key];
```

## 读取图片功能扩展

此处主要涉及到的可能的技术点:

### 1. 如何用基础的网络类库下载文件?

依然是要封装一个简单函数,下载完成后,通过block传出文件临时的保存位置:

```objc
-(void) downloadFileFrom:(NSString * ) urlStr completionHandler: (void (^)(NSURL *location, NSError * error)) completionHandler
{
    NSURL * url = [NSURL URLWithString:urlStr];

    NSURLSessionConfiguration * defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession * defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate:self delegateQueue: [NSOperationQueue mainQueue]];

    NSURLSessionDownloadTask * downloadTask =[ defaultSession downloadTaskWithURL:url
                                                                completionHandler:^(NSURL * location, NSURLResponse * response, NSError * error)
                                              {

                                                  completionHandler(location,error);

                                              }];
    [downloadTask resume];

}
```

### 2. 如何判断bundle中是否含有某文件?

可以使用 *fileExistsAtPath*,但其实使用 *-pathForResource: ofType:* 就够了,因为找不到资源问加你时,它返回nil,所以我们直接调用它,然后判断返回是否为 *nil* 即可:

```objc
NSString * imgPath = [mainBundle pathForResource:imgName ofType:@"png"];
```
### 3. 将代码如何与原有的imageNamed:逻辑合并?

不需要初始复制到缓存目录 + 初始请求最新的资源补丁信息 + 代码迁移合并 + 接口优化

## 相对完整的逻辑代码

注意,按照目前的设计,就不需要初始把原来ipa中的bundle复制到缓存目录了;当缓存目录中没有相关资源时,会自动尝试从ipa中的bundle读取,bundle约定统一使用 main.bundle 来简化操作,

类目,对外暴露两个方法:

```objc
#import <UIKit/UIKit.h>

@interface UIImage (imageNamed_bundle_)
/* load img smart .*/
+ (UIImage *)yf_imageNamed:(NSString *)imgName;

/* smart update for patch */
+ (void)yf_updatePatchFrom:(NSString *) pathInfoUrlStr;
@end
```

App启动时,或在其他合适的地方,要注意检查有无更新:

```objc
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
    // Override point for customization after application launch.

    /* fetch pathc info every time */
    NSString * patchUrlStr = @"https://raw.githubusercontent.com/ios122/ios_assets_hot_update/master/res/patch_04.json";
    [UIImage yf_updatePatchFrom: patchUrlStr];

    return YES;
}
```

内部实现,优化了许多,但也算不上复杂:

```objc
#import "UIImage+imageNamed_bundle_.h"
#import <SSZipArchive.h>

@implementation UIImage (imageNamed_bundle_)

+ (NSString *)yf_sourcePatchKey{
    return @"SOURCE_PATCH";
}

+ (void)yf_updatePatchFrom:(NSString *) pathInfoUrlStr
{
    [self yf_fetchPatchInfo: pathInfoUrlStr
       completionHandler:^(NSDictionary *patchInfo, NSError *error) {
           if (error) {
               NSLog(@"fetchPatchInfo error: %@", error);
               return;
           }

           NSString * urlStr = [patchInfo objectForKey: @"url"];
           NSString * md5 = [patchInfo objectForKey:@"md5"];

           NSString * oriMd5 = [[[NSUserDefaults standardUserDefaults] objectForKey: [self yf_sourcePatchKey]] objectForKey:@"md5"];
           if ([oriMd5 isEqualToString:md5]) { // no update
               return;
           }

           [self yf_downloadFileFrom:urlStr completionHandler:^(NSURL *location, NSError *error) {
               if (error) {
                   NSLog(@"download file url:%@  error: %@", urlStr, error);
                   return;
               }

               NSString * patchCachePath = [self yf_cachePathFor: md5];
               [SSZipArchive unzipFileAtPath:location.path toDestination: patchCachePath overwrite:YES password:nil error:&error];

               if (error) {
                   NSLog(@"unzip and move file error, with urlStr:%@ error:%@", urlStr, error);
                   return;
               }

               /* update patch info. */
               NSString * source_patch_key = [self yf_sourcePatchKey];
               [[NSUserDefaults standardUserDefaults] setObject:patchInfo forKey: source_patch_key];
           }];
       }];

}

+ (NSString *)yf_relativeCachePathFor:(NSString *)md5
{
    return [@"patch" stringByAppendingPathComponent:md5];
}

+ (UIImage *)yf_imageNamed:(NSString *)imgName{
    NSString * bundleName = @"main";

    /* cache dir */
    NSString * md5 = [[[NSUserDefaults standardUserDefaults] objectForKey: [self yf_sourcePatchKey]] objectForKey:@"md5"];

    NSString * relativeCachePath = [self yf_relativeCachePathFor: md5];

    return [self yf_imageNamed: imgName bundle:bundleName cacheDir: relativeCachePath];
}

+ (UIImage *)yf_imageNamed:(NSString *)imgName bundle:(NSString *)bundleName cacheDir:(NSString *)cacheDir
{
    NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);

    bundleName = [NSString stringWithFormat:@"%@.bundle",bundleName];

    NSString * ipaBundleDir = [NSBundle mainBundle].resourcePath;
    NSString * cacheBundleDir = ipaBundleDir;

    if (cacheDir) {
        cacheBundleDir = [[[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches"] stringByAppendingPathComponent:cacheDir];
    }

    imgName = [NSString stringWithFormat:@"%@@3x",imgName];

    NSString * bundlePath = [cacheBundleDir stringByAppendingPathComponent: bundleName];
    NSBundle * mainBundle = [NSBundle bundleWithPath:bundlePath];
    NSString * imgPath = [mainBundle pathForResource:imgName ofType:@"png"];

    /* try load from ipa! */
    if ( ! imgPath && ! [ipaBundleDir isEqualToString: cacheBundleDir]) {
        bundlePath = [ipaBundleDir stringByAppendingPathComponent: bundleName];
        mainBundle = [NSBundle bundleWithPath:bundlePath];
        imgPath = [mainBundle pathForResource:imgName ofType:@"png"];
    }

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

+ (void)yf_fetchPatchInfo:(NSString *) urlStr completionHandler:(void (^)(NSDictionary * patchInfo, NSError * error))completionHandler
{
    NSURLSessionConfiguration *defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate: nil delegateQueue: [NSOperationQueue mainQueue]];

    NSURL * url = [NSURL URLWithString:urlStr];

    NSURLSessionDataTask * dataTask = [defaultSession dataTaskWithURL:url
                                                    completionHandler:^(NSData *data, NSURLResponse *response, NSError *error) {
                                                        NSDictionary * patchInfo = [NSJSONSerialization JSONObjectWithData:data options:0 error:&error];;

                                                        completionHandler(patchInfo, error);
                                                    }];

    [dataTask resume];
}

+ (void) yf_downloadFileFrom:(NSString * ) urlStr completionHandler: (void (^)(NSURL *location, NSError * error)) completionHandler
{
    NSURL * url = [NSURL URLWithString:urlStr];

    NSURLSessionConfiguration *defaultConfigObject = [NSURLSessionConfiguration defaultSessionConfiguration];
    NSURLSession *defaultSession = [NSURLSession sessionWithConfiguration: defaultConfigObject delegate:nil delegateQueue: [NSOperationQueue mainQueue]];

    NSURLSessionDownloadTask * downloadTask =[ defaultSession downloadTaskWithURL:url
                                                                completionHandler:^(NSURL *location, NSURLResponse *response, NSError *error)
                                              {

                                                  completionHandler(location,error);

                                              }];
    [downloadTask resume];
}

+ (NSString *)yf_cachePathFor:(NSString * )patchMd5
{
    NSArray * LibraryPaths = NSSearchPathForDirectoriesInDomains(NSLibraryDirectory, NSUserDomainMask, YES);

    NSString * cachePath = [[[LibraryPaths objectAtIndex:0] stringByAppendingFormat:@"/Caches"] stringByAppendingPathComponent:[self yf_relativeCachePathFor: patchMd5]];

    return cachePath;
}

@end
```

现在,加载图片的代码更简单了:

```objc
UIImage * image = [UIImage yf_imageNamed:@"sub/sample"];
self.sampleImageView.image = image;
```

如果热更新生效,运行看到的应该是一个拍卖图片:


![热更新生效](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_4/2.png?raw=true)

## 后记

我觉得,这篇文章最大的特点是,完整记录了一次优化解决问题的过程;示例代码看起来前后有些不太统一,是因为: 我不是先有了方案再写博客,而是借助博客本身来梳理思路,简化逻辑!如此,写博客,就不单单是一个耗时的分享知识的过程,更成为了一个帮助自己思考的有力工具!赞!!!

## 参考资源:

* [本节内容完整可执行Xcode工程代码,不到100k](https://github.com/ios122/ios_assets_hot_update/raw/master/res/ios_assets_hot_update_4.zip)
* [系列文章,专属github项目](https://github.com/ios122/ios_assets_hot_update)
* [iOS NSURLSession Example (HTTP GET, POST, Background Downlads )](http://hayageek.com/ios-nsurlsession-example/)
* [价值100W的经验分享: 基于JSPatch的iOS应用线上Bug的即时修复方案,附源码.](http://www.ios122.com/2015/12/jspatconline/)
* [ZipArchive is a simple utility class for zipping and unzipping files on iOS and Mac.](https://github.com/ZipArchive/ZipArchive)
* [pod 的安装和使用](http://www.jianshu.com/p/6d1bc61b11b5)
