# 实现iOS图片等资源文件的热更新化(五): 一个简单完整的资源热更新页面

## 简介

![更新结果](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_5/01.png?raw=true)


一个简单的关于页面,有一个图片,版本号,App名称等,着重演示各个系列的文章完整集成示例.

## 动机与意义

这是系列文章的最后一篇.今天抽空写下,收下尾.文章本身会在第四篇的基础上,简单扩充下代码,实现在线下载与重置更改的功能.

如果能较为仔细地阅读前四篇文章,第五篇给出的示例,应当是可以理解为无足轻重的.但是,大多数时候,我们更多的可能只是需要一个简易的解决方案,就是那种拿来就可以用的东西,那种我们需要先能看到一个简要的示例来看下效果再解决是否再继续阅读的方案.如此,对于很久以后,由于各种原因被搜索引擎或者其他文章的链接导向此系列文章的人来说,他们可能更想看到一个简要的示例,来决定系列的文章,在他们那个时间点,是否依然有意义.

截止目前而言,我对博客记录本身的定位,依然是属于一个辅助思考的工具.当你看到这篇文章的时候,可能你已经在用Xcode9 Xcode10了,可能代码示例都已经跑不起来了,但是我相信每篇文章所展示的那些参考链接和本身所透漏出的某些思考,或许对于你仍然是有某种启发的.

## 思路与实现

1. App版本和名称,可以直接读取;
2. 在线下载更新资源,可以借助前一篇的代码实现;
3. 重置的话,可以选择清除补丁信息或者直接清除补丁,本文选择第一种;

### 核心代码:

我需要先扩展下更新资源的方法,使其在更新完整后,能返回更新的结果,以便于我进行进一步的操作,如重新显示某个图片:

```objc
+ (void)yf_updatePatchFrom:(NSString *) pathInfoUrlStr completionHandler:(void (^)(BOOL success, NSError * error))completionHandler
{
    if ( ! completionHandler) {
        completionHandler = ^(BOOL success, NSError * error){
            // nothing to do...
        };
    }

    [self yf_fetchPatchInfo: pathInfoUrlStr
       completionHandler:^(NSDictionary *patchInfo, NSError *error) {
           if (error) {
               NSLog(@"fetchPatchInfo error: %@", error);
               completionHandler(NO, error);
               return;
           }

           NSString * urlStr = [patchInfo objectForKey: @"url"];
           NSString * md5 = [patchInfo objectForKey:@"md5"];

           NSString * oriMd5 = [[[NSUserDefaults standardUserDefaults] objectForKey: [self yf_sourcePatchKey]] objectForKey:@"md5"];
           if ([oriMd5 isEqualToString:md5]) { // no update
               completionHandler(YES,nil);
               return;
           }

           [self yf_downloadFileFrom:urlStr completionHandler:^(NSURL *location, NSError *error) {
               if (error) {
                   NSLog(@"download file url:%@  error: %@", urlStr, error);
                   completionHandler(NO, error);
                   return;
               }

               NSString * patchCachePath = [self yf_cachePathFor: md5];
               [SSZipArchive unzipFileAtPath:location.path toDestination: patchCachePath overwrite:YES password:nil error:&error];

               if (error) {
                   NSLog(@"unzip and move file error, with urlStr:%@ error:%@", urlStr, error);
                   completionHandler(NO, error);
                   return;
               }

               /* update patch info. */
               NSString * source_patch_key = [self yf_sourcePatchKey];
               [[NSUserDefaults standardUserDefaults] setObject:patchInfo forKey: source_patch_key];
               completionHandler(YES,nil);
           }];
       }];

}
```
然后是一个自定义的在线更新的点击方法:

```objc
- (IBAction)onlineUpdate:(id)sender {
    __weak ViewController * weakSelf = self;
    [UIImage yf_updatePatchFrom:@"https://raw.githubusercontent.com/ios122/ios_assets_hot_update/master/res/patch_04.json" completionHandler:^(BOOL success, NSError *error) {
        UIImage * image = [UIImage yf_imageNamed:@"sub/sample"];
        weakSelf.sampleImageView.image = image;
    }];
}
```

还需要一个自定义的reset方法,考虑到以后的扩展性和目前的需要,使其支持block传出操作结果:

```objc
+ (void )yf_reset:(void (^)(BOOL success, NSError * error))completionHandler
{
    if ( ! completionHandler) {
        completionHandler = ^(BOOL success, NSError * error){
            // nothing to do...
        };
    }

    [[NSUserDefaults standardUserDefaults] setObject:nil forKey: [self yf_sourcePatchKey]];
    completionHandler(YES, nil);
}
```

![重置](https://github.com/ios122/ios_assets_hot_update/blob/master/imgs/ios_assets_hot_update_5/02.png?raw=true)

具体使用起来,就很简单,重置后,更新下图片即可:

```objc
- (IBAction)reset:(id)sender {
    __weak ViewController * weakSelf = self;

    [UIImage yf_reset:^(BOOL success, NSError *error) {
        if (success) {
            UIImage * image = [UIImage yf_imageNamed:@"sub/sample"];
            weakSelf.sampleImageView.image = image;
        }else
        {
            NSLog(@"reset error:%@", error);
        }
    }];
}
```

## 系列文章心得小结

这是第二个系列文章."我们应该相信大多数人们对于美好的东西是有鉴赏的能力" -- 如果能在这一点上达成共识,下面我说的,或许值得继续一读:

### 一些数据

* [开源中国](https://my.oschina.net/ios122/blog) 推荐了2篇博客: 已发布的四篇系列文章,有两篇在OSC上获得了小编的全站推荐.
* [简书](http://www.jianshu.com/users/2b6a83ce2876/latest_articles) 首页推荐两篇,获得打赏一次: 简书本身的技术属性,可能算不上很强,但近来搜索技术资料时,有好多都链接指向简书,而且信息大都很及时很新鲜,原因未知,所以最近自己也开始同步在简书更新文章,至于收到打赏,其实就只有2元的辣条钱,但是很明显这个童鞋是搜索某个信息时,被导向了我的文章,而且从其评论来看,确实对其有一定的帮助 -- 我觉得,能够被需要的人看到,这才是最让博主开心的事!
* [微博](http://weibo.com/ios122) 相关博文有3位大V转发: 某种程度上,我觉得这算是一种认可.不过,我本身其实并不怎么玩微博;微博的信息太容易被淹没,但如果只考虑传播属性的话,微博的扩散效果其实是极好的.
* [segmentfault](https://segmentfault.com/blog/ios122) 把文章添加到头条之后,被segmentfault的CEO [高阳Sunny ](https://segmentfault.com/u/sunny) 点赞了两次,微博转发一次.有种受宠若惊的感觉,不过后来我想可能人家更多地只是想推一下新出的"头条"功能.
* [csdn](http://blog.csdn.net/column/details/ios122.html)首页推荐一次: 通知原文是,"你的实现iOS图片等资源文件的热更新化(四): 一个最小化的补丁更新逻辑被推荐到首页喽！" 这个确实挺值得纪念的!刚开始的时候,我发篇文章,如果有外链,CSDN就必须要审查之后才能被公开看到!

### 几点心得

* 工作第一,博客分享第二: 我不指望能将来靠博客挣稿费,那也就意味着工作上的事务永远都必须是优先处理的.所以,博客的更新时间并不能真正固定.还有就是,不希望博客分享本身成为一种负担,如果实在没心情或者生活中有其他事的话,我也就真的搁在那,以后再写.
* 不要被以前的主题束缚,写自己真正需要或者真正感兴趣的:这个系列,从时间上来说,确实比预期的一周迟了一个月;但是从实际效果来看,要比上一个Spark系列好很多.但是当初决定这个系列的内容时,我也是很纠结,是要继续Spark大数据题材,还是分享下自己一直想深入研究,却一直抽不出时间的资源包优化问题.最终,还是选择了后者,因为目前对Spark需要的场景,在自己工作中确实不多.
* 记录思路和参考资源,可能比解决方案本身更重要:更多的,是阅读其他人博客的经验;遇到完全一致的问题的可能性很小,而且许多情况下,是从博主的相关引用中关于类似问题更细节的参考中,找到答案的;另外,各种引用资料,可能也给人一种很高大上的感觉.
* 你需要的时间比你预期的要更长: 你以为半个小时可以搞定的文章,可能会花费两个小时,才勉强收尾;你以为很简答的一个技术点,在某个细节上演绎之后,可能会比你想象中更经验.当你意识到,自己正在做的东西,是会被大家公开阅读和鉴赏时,你会不由自主地想多做一点,多查一些,多优化一点,不想显得太low.

### 小规划

* 题材,坚持系列文章: 我发现系列文章,真的有利于帮助自己进行和坚持深入地有序思考.
* 主题,确定为移动混合开发:最近一年都在用[ReactNative](http://facebook.github.io/react-native/docs/getting-started.html)开发App,但是单纯地使用,已经不能满足我了,我想深入研究下内部地某些实现机制.作为对比,会研究下勉强算是社区驱动的[Weex](http://alibaba.github.io/weex/);另外,还会关注下国内的商业驱动的[APICloud](http://www.apicloud.com)平台.
* 内容会涉及iOS,Android,HTML5和自动化脚本: iOS算是本职工作,Android和HTML是自己迫切需要补上的技能,而自动化脚本的编写能力将在很大程度上决定自己自动处理复杂信息的能力和未来的发展 -- 都说Lisp是宇宙第一语言,但目前还是基础的shell脚本用的比较多.
* 文章和评论宜只谈技术: ReactNative 所代表的混合开发的方向,在一定程度上已经获得了国内以BAT为代表的一线技术公司的认可,大家可以去[showcase示例](http://facebook.github.io/react-native/showcase.html)具体看下;Weex,目前只是粗读了下文档,三端公用代码,确实有些脑洞,其内部实现应该具有相当程度的学习价值,但其理念不敢苟同,3端共用代码,意味着要取三端各自平台优势的交集,可能也就意味着要牺牲3个平台的各自的独特性和优势 -- 如果真的是这这样,那ReactNative,也是可以自称"一处编写,处处运行"的;APICloud,商业驱动,从产品角度来说,较为完善,混合开发只是服务的一部分,按照目前的发展路线,如果未来HTML发展再迅速一点,或许会有极大出线的可能.

## 参考资源

* [本节内容完整可执行Xcode工程代码,不到0.5M](https://github.com/ios122/ios_assets_hot_update/raw/master/res/ios_assets_hot_update_5.zip)
* [系列文章专属github项目](https://github.com/ios122/ios_assets_hot_update)
* [CFBundleDisplayName returns 'null'](http://stackoverflow.com/questions/18989045/cfbundledisplayname-returns-null)
