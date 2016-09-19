# 实现iOS图片等资源文件的热更新化(一): 从Images.xcassets导出合适的图片

本文会基于一个已有的脚本工具自动导出所有的图片;最终给出的是一个从 Images.xcassets 到基于文件夹的精简 合适 的图片资源集的完整过程.难点在于从完整图片集到精简图片集,肯定是基于一个定制化的脚本,自定义导出的.如果自己手动导出?那可有的忙喽~

## Images.xcassets 与 Assets.car

Images.xcassets,是Xcode项目中的,用于存放资源文件.那么我们为什么不直接处理 Images.xcassets 呢?因为Images.xcassets中存放的图片名称可能与图片的资源名称不一致,最终决定图片资源名的是资源文件夹的名称;也有可能Images.xcassets存放的是pdf格式的图片,这样可以自动预编译对应尺寸的图片资源.

Images.xcassets 编译后,最终ipa包中,是以Assets.car包的形式出现的,内部是处理后的图片名.此处的文件名与我们代码中引用的图片资源名称是一致的.

也就是说: 直接基于Assets.car进行处理,可以使我们的使用图片处的代码变更尽可能少.

## 使用 cartool 从 Assets.car 导出图片

Assets.car 无法直接zip解压,需要借助专门的工具,此处推荐: [cartool](https://github.com/steventroughtonsmith/cartool) 使用方法,参见: [iOS学习之解压Assets.car](http://www.jianshu.com/p/a5dd75102467)

如果你缺少足够复杂的Assets.car或者cartool用法有问题,可以直接使用我处理过的资源:[https://github.com/ios122/ios_assets_hot_update/tree/master/res](https://github.com/ios122/ios_assets_hot_update/tree/master/res)

针对文章github给定的目录, cartool的用法,可以简述为:
cd 到 res目录,然后

```sh
mkdir Assets
./cartool  ./Assets.car ./Assets
```

## 其实使用一张图片就可以额兼容iPhone/iPad

从 Assets.car 导出后的图片,大致有以下几种:

* 只存在@1x的图: 如 2.png
* 只存在@1x和@2x的图: 如 account.png 和 account@2x.png
* 只存在@2x的图: 如add-1@2x.png
* 只存在@2x与@3x的图片: 如 10@2x.png 和 10@3x.png
* 同时存在三种尺寸的图片: 如 1.png 1@2x.png 和 1@3x.png
* 区分iphone与ipad的图片,此类图一般由pdf自动在预编译时生成: 如bg_mypage_edit~ipad.png bg_mypage_edit~ipad@2x.png bg_mypage_edit~ipad@3x.png bg_mypage_edit~iphone.png bg_mypage_edit~iphone@2x.png bg_mypage_edit~iphone@3x.png
* 汉语命名的图片: 如 提醒.png

以上图片的原因,很大一部分是由于App迭代引起的.对于一个图片,存在上述不同情况时,图片通常加载与当前屏幕比例(scale)最符合的图片,具体细节下一篇文章会更完整描述.

经过我自己的实验与网上各种资料的查询,使用 @3x 的图片是可以同时作为 iPhone和iPad的通用图标的.当然,这是需要自定义 imageNamed方法,也是下一篇文章的重点. 2套共5个图片,现在只需要1个图片,理论图片资源体积可以减小
((1 + 2 + 3 + 3 + 1.5) - 3) / (1 + 2 + 3 + 3 + 1.5) = 71.428571 %  (信息量超大的速算法,看不懂就当是个冷笑话吧~\(≧▽≦)/~)

## 自动归类脚本思路

我们想要获取的是 *可用的@3x图片文件夹* 与 *不包含@3x图片的有问题的资源列表*. 对于不存在@3x副本的图片,很大可能这个资源已经被废弃了.这一块,暂定手动去排查与核实.如果一个图片仍在使用但是不存在@3x的副本,绝对是RD挖了一个坑,等你来填!

基本思路是:

1. 去除 ~ipad 结尾的图片,如bg_mypage_edit~ipad.png;
2. 去除 ~iphone 图片中的 ~iphone文字,如bg_mypage_edit~iphone@3x.png 重命名为 bg_mypage_edit@3x.png;
3. 将含有@3x的图片组的@1x @2x @3x 的图片按顺序移动到单独文件夹 如 assets_3x,并都命名为@3x,此时原文件夹中即为有问题的资源,新文件夹中为有效的资源文件,且只保留了@3x;
4. 将原资源文件夹命名为assets_error,以供以后使用;
5. 人工确认非法图片是否具有存在意义,存在则寻找其@3x副本放到 assets_3x 文件夹;

## 自动归类脚本实现

除了以上的第五步以外,前四步都可以自动化运行:

```sh
#0. 需要先cd到解压后的Assets目录;
#1. 去除 ~ipad 结尾的图片,如bg_mypage_edit~ipad.png;
find . -iname "*~ipad*.png" -delete

#2. 去除 ~iphone 图片中的 ~iphone文字;
find . -name "*~iphone.png" -exec sh -c 'for i do mv -- "$i" "${i%~iphone.png}.png"; done' sh {} +

find . -name "*~iphone@2x.png" -exec sh -c 'for i do mv -- "$i" "${i%~iphone@2x.png}@2x.png"; done' sh {} +

find . -name "*~iphone@3x.png" -exec sh -c 'for i do mv -- "$i" "${i%~iphone@3x.png}@3x.png"; done' sh {} +

# 3.将含有@3x的图片组的@1x @2x @3x 的图片按顺序移动到单独文件夹 如 assets_3x,并都命名为@3x,此时原文件夹中即为有问题的资源,新文件夹中为有效的资源文件,且只保留了@3x;

mkdir ../assets_3x

find . -name "*@3x.png" -exec sh -c 'for i do mv -- "${i%@3x.png}.png" "../assets_3x/${i%@3x.png}@3x.png"; mv -- "${i%@3x.png}@2x.png" "../assets_3x/${i%@3x.png}@3x.png";mv -- "${i%@3x.png}@3x.png" "../assets_3x/${i%@3x.png}@3x.png";done' sh {} +

# 4.将原资源文件夹命名为assets_error,以供以后使用;
cd ..
mv Assets assets_error
```

最终得到的 assets_3x 即为可用资源,assets_error 即为需要手动确认可用性的资源.

## 收获与感悟:

1. 项目中,图片这一块,的确有许多无用的或不合理的资源,需要及早解决;
2. shell 脚本是基于路径进行复制,移动等操作的,如 find的结果,其实是一个文件路径,借助它,提出了一个简单的区分可用于不可用资源的方法;
3. 写博客,确实可以使思路更清晰有序,坦白讲,这本来是一个我不敢碰的优化任务,一个一个比对,想想都头大.最终的处理结果,还是给出了一定数量的无用图片,但是我根据其名字就可以确定其位置,非常好处理了,已经省了不少功夫了;而且,要比我手动排查地可信多了.

---
系列专属github地址: [https://github.com/ios122/ios_assets_hot_update](https://github.com/ios122/ios_assets_hot_update)
