---
title: 在你的hexo中也放入一只bilibili娘吧！
date: 2019-04-15 16:20:09
tags: 
  - hexo
  - live2d
---



## 前言

因为突然想要在自己的hexo站点里也放入一只看板娘，于是开始搜集资料，期间遇到一些问题花了点时间解决，由于前端实在是在苦手（后端程序员的苦笑）太难受了。。

所以决定开始使用Hexo开始写博客了，总不能一直挂着初始化的[Hello World](/2019/04/10/hello-world/)一直挂在首页吧哈哈哈，所以打算记录一下这个过程，假如也有人遇到同样的问题可以方便解决一下（如果能看到的话

## 目的

前面说的都是废话了，忘了说我的目的了。

主要目的在于使用[hexo-helper-live2d](https://github.com/EYHN/hexo-helper-live2d)放一只看板娘，使用[bilibili-haruna](https://github.com/52cik/bilibili-haruna)33娘的模型

嘛不废话了，开始了～



## 开始



### 安装 hexo-helper-live2d

如果没有安装`hexo-helper-live2d`的话需要先安装啦

```
npm install --save hexo-helper-live2d
```

但是我在安装的时候提示下面的内容，不知道干啥用的为什么的，但是我还是安装了下面的那个东西了，如果你没有碰到这个的话那就直接跳过了

```
npm WARN babel-eslint@10.0.1 requires a peer of eslint@>= 4.12.1 but none is installed. You must install peer dependencies yourself.
```



### 模型

如果就执行上面的命令就可以的话就完全没有必要记录这次过程了。因为添加bilibili娘模型需要手动添加，没法全自动呢。

大体的步骤如下

1. 在你的博客根目录下创建`live2d_models`文件夹
2. 在`live2d_models`文件夹下创建一个将要添加的模型的名字的文件夹，如我这里用33娘为例，创建名为`33`的文件夹, 路径为`live2d_models/33`
3. 在`33`目录下创建两个文件夹`moc` `mtn`
4. 在`moc`文件夹下创建`33.1024`
5. 将[bilibili-haruna](https://github.com/52cik/bilibili-haruna)里面的模型下载回来
6. 根据上面下载回来你想要的模型里文件一一放进去我们创建好的文件夹
7. 因为需要一个`33.model.json`文件在`33`文件夹的目录下，找到你想要的那个主题的文件复制过来然后改一下名，比如我选择的主题是`2017valley`，所以将文件`model.2017.valley.json`复制过来并改名为`33.model.json`
8. 需要修改第7步中的`33.model.json`文件



现在你的文件夹结构大概为这样，先把目录放出来之后再详细说一下上面中有一些细节步骤

![image-20190415165616631](/images/201904/fileconstruct.png)

确认以上目录结构没有问题之后就可以继续了（主要是里面的文件夹的目录结构，文件内容暂时不用管，后续文章中会说明的。）



现在回到第6步中。

> 比如：我怎么知道我要复制什么文件进去呢？

因为我在查看提供npm安装的模型github仓库[live2d-widget-models](https://github.com/xiazeyu/live2d-widget-models)中查看目录结构，如果有兴趣可以点进去该仓库中查看，里面提供了挺多模型的。

我这里就直接放出里面某个模型的结构图了，该结构如下

![image-20190415170643812](/images/201904/widget-models-file-construct.png)

之后我就按照这个目录结构在`第5步`中下载好的模型文件一一找到并复制进在`1-4步`中创建的文件夹中。

以下图是我们下载回来的bilibili娘模型的仓库，红色框框住的是我们目前需要的文件

![image-20190415171548522](/images/201904/bili33-require-file.png)

根据对应的文件格式放到对应的文件夹中。



复制好了之后，打开`33.model.json`文件，将该文件修改成如下内容。（主要修改的是文件的相对位置）

但是在下面的内容中，你应该是没有`"moc/33.1024/texture_00.png"`这个的。根据文件中的`"33.1024/closet.default.v2/texture_00.png"`找到那个`texture_00.png`文件，并复制到`moc`文件夹下。

下面是`33.model.json`文件：

```json
{
  "type": "Live2D Model Setting",
  "name": "33",
  "label": "33",
  "model": "moc/33.v2.moc",
  "textures": [
    "moc/33.1024/texture_00.png",
    "moc/33.1024/texture_01.png",
    "moc/33.1024/texture_02.png",
    "moc/33.1024/texture_03.png"
  ],
  "layout": { "center_x": 0, "center_y": 0.1, "width": 2.3, "height": 2.3 },
  "motions": {
    "idle": [
      {
        "file": "mtn/33.v2.idle-01.mtn",
        "fade_in": 2000,
        "fade_out": 2000
      },
      {
        "file": "mtn/33.v2.idle-02.mtn",
        "fade_in": 2000,
        "fade_out": 2000
      },
      {
        "file": "mtn/33.v2.idle-03.mtn",
        "fade_in": 100,
        "fade_out": 100
      }
    ],
    "tap_body": [
      {
        "file": "mtn/33.v2.touch.mtn",
        "fade_in": 500,
        "fade_out": 200
      }
    ],
    "thanking": [
      {
        "file": "mtn/33.v2.thanking.mtn",
        "fade_in": 2000,
        "fade_out": 2000
      }
    ]
  }
}

```

到这里应该`live2d_models`文件夹下的文件应该都处理完毕了，接下来将配置`_config.yml`文件，据说是根目录下的`_config.yml`或者是主题下的都行，我没试过在主题下的配置文件，我直接在根目录的配置文件的。

配置内容大概如下：

```yaml
live2d:
  enable: true
  # 这里设置成local
  scriptFrom: local
  model:
    use: "33"
    scale: 1
    hHeadPos: 0.5
    vHeadPos: 0.618
  display:
    superSample: 2
    width: 150
    height: 300
    position: left
    hOffset: 0
    vOffset: -20
  mobile:
    # 因为我没有这个需求，我就直接false了
    show: false
    scale: 0.5
  react:
    opacityDefault: 0.7
    opacityOnHover: 0.2

```

这里不贴各个配置项的具体内容了，大家可以到[https://github.com/EYHN/hexo-helper-live2d/blob/master/README.zh-CN.md#详细的设置](https://github.com/EYHN/hexo-helper-live2d/blob/master/README.zh-CN.md#%E8%AF%A6%E7%BB%86%E7%9A%84%E8%AE%BE%E7%BD%AE)看，再根据自己需求改就行了，这篇文章讲到能跑起来为止就ok了



到目前为止应该是能跑起来了，不信你看左手边233



## 参考

感谢以下项目及其作者提供了可爱的看板娘～

[hexo-helper-live2d](https://github.com/EYHN/hexo-helper-live2d/blob/master/README.zh-CN.md) — by EYHN

[bilibili-haruna](<https://github.com/52cik/bilibili-haruna>) — by 52cik

[live2d-widget-models](<https://github.com/xiazeyu/live2d-widget-models>) — by xiazepu



感谢以下作者提供优秀的教程供参考

[在你的 Hexo 小站中放一只看板娘！](https://plxus.github.io/2018/03/%E5%9C%A8%E4%BD%A0%E7%9A%84-Hexo-%E5%B0%8F%E7%AB%99%E4%B8%AD%E6%94%BE%E4%B8%80%E5%8F%AA%E7%9C%8B%E6%9D%BF%E5%A8%98%EF%BC%81/)  — by plxus

[hexo 添加live2d看板动画](<https://juejin.im/post/5b447600f265da0fa332d8ac>) — by annkee

