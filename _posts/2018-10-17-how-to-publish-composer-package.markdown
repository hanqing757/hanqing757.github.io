---
layout: post
title:  "如何把自己的项目发布为composer包"
date:   2018-10-17 20:28:00
categories: composer
tags: composer
excerpt: 如何将php项目发布为composer包
mathjax: true
---

* content
{:toc}

Composer是PHP的依赖包管理工具，我们不仅要会使用Composer获取别人的包，也要学会制作和分享自己的Composer包，下面分享自己发布Composer包的过程。

首先需要有一个完善的可供使用的包（PHP项目）,为了发布自己的Composer包第一步先完善composer.json文件
```php
{
    "name":"luffy/randomizer",
    "type":"library",
    "description":"get random data from random.org",
    "keywords":["random"],
    "authors":[
        {
            "name":"Qing Han",
            "email":"hanqing002@ke.com"
        }
    ],
    "require":{
        "php": "^5.6 || ^7.0",
        "guzzlehttp/guzzle" : "^6.3"
    },
    "repositories":{
         "packagist": {
                "type": "composer",
                "url": "https://packagist.phpcomposer.com"
            }
    },
    "minimum-stability": "stable",
    "autoload": {
        "psr-4": {
            "luffy\\": "src/"
        }
    }
}
```
name表示包的名字，composer会通过这个名字搜索包。type表示包的安装类型，默认是library。autoload表示包的自动载入方式，常用的有psr0和psr4，后面会详细分析这两种载入方式的不同。
在github上新建仓库，并将本地包推送到远程仓库。然后给项目添加release版本号，以便require可以根据版本号获取响应的包。
登录[packagist.org](https://packagist.org/),点submit，输入github项目的地址，如https://github.com/hanqing757/trueRandom，网站会抓取你的项目页面展示如下
右边会显示你release的版本号，这就表明已经将你的项目发布到了packagist上了。接下来本地新建一个项目，编辑composer.json文件
```php
{
	"require":{
        "luffy/randomizer" : "^1.2",
        "php": "^5.6 || ^7.0"
    },
    "repositories":[
        {
                "type": "composer",
                "url": "https://packagist.phpcomposer.com"
        }
    ]
}
```
若你的项目没有发布到packagist.org上，repositories可以这么写
```php
"repositories":[
        {
                "type": "vcs",
                "url": "https://github.com/hanqing757/trueRandom"
        }
    ]
```
url为对应github项目地址。当然一般会将包放在packagist.org上，此时repositories中的type是composer，对应的url是composer的中国镜像，会加快包的下载速度。
对于autoload，psr0方式和psr4方式都定义了命名空间到路径的映射，命名空间要以双反斜杠\\结尾以精确匹配，否则Foo会匹配到Foobar。psr0中的类名中如果有下划线则会自动转换成路径分割符，并且映射方式也不一样，psr0路径更深比如
```php
"autoload": {
        "psr-0": {
            "luffy\\": "src/"
        }
    }
```
当遇到luffy\random\Randomizer.php时会加载src/luffy/random/Randomizer.php，使用psr4加载时
```php
"autoload": {
        "psr-4": {
            "luffy\\": "src/"
        }
    }
```
当遇到luffy\random\Randomizer.php时会加载src/random/Randomizer.php，命名空间不会展示在路径中。composer加载psr0时键值映射存放在
```php
vendor/composer/autoload_namespaces.php
```
加载psr4时键值映射存放在
```php
vendor/composer/autoload_psr4.php
```
这时候在你的项目根目录下执行composer install即可下载luffy/randomizer 的最新版本。