---
layout: post
title:  "并发请求之Yar和curl"
date:   2019-1-7 15:00:00
categories: concurrent
tags: concurrent-requests yar curl
excerpt: 并发请求之Yar和curl
mathjax: true
---

* content
{:toc}

前一阵做的项目是为移动端提供API接口，对于移动端来讲，通常一个页面对应一个后端的接口，但有时候也会出现一个页面对应后端很多个接口，比如一个房源详情页涉及的数据比较多的时候，那么作为API层来讲，应该给客户端提供尽量简洁的接口和数据，在一个接口中需要将后端提供的众多接口进行组合，将数据整理好之后返给客户端。为了尽量减少接口请求时间，在接口之间没有依赖关系的情况下，可以对这些接口进行并发处理。目前在项目中使用了两种并发请求的方式，一个是[Yar](http://php.net/manual/zh/book.yar.php)，这是鸟哥写的一个php扩展，另一种就是php支持的curl请求。

Yar是一个轻量级的并行的rpc框架，使用的时候首先需要注册server端
```php
class API{
    public function api1($params){
    } 
    public function api2($params){
    }
    public function api3($params){
    }
}

$service = new Yar_Server(new API());
$service->handle();
```
调用handle函数，就表示启动了服务，这样就可以接收来自客户端的请求了。
客户端的并行调用方法如下
```php
Yar_Concurrent_Client::call("http://host/api/", "api1", array("parameters"), "callback");
Yar_Concurrent_Client::call("http://host/api/", "api2", array("parameters"), "callback");
Yar_Concurrent_Client::call("http://host/api/", "api3", array("parameters"), "callback");
Yar_Concurrent_Client::loop(); //send

function callback($retval,$callinfo){
    if($callinfo === null){
        return;
    }
    $this->data[$callinfo['method']] = $retval;
}
```
其中的call方法表示注册一个远程服务调用，但并不会实际的去发请求，直到下面的loop方法被调用。每完成一次请求调用，它的callback方法就会被执行，在所有的请求调用结束后，Yar会再调用一次callback，只不过这次调用的callback中的callinfo为空，这样根据callinfo为空就可以判断调用结束。我们将方法名和相应的返回组成一个关联数组进行下一步的处理。
通常我们会将Yar的server端作为单独的一部分写在controller层，那么在并行调用的时候实际上会回本机重新请求，因此我们看到这种方式比直接并行请求会多消耗一些时间。并且如果我们的请求需要设置一些header时，Yar无法做到（2.0.4版本才可以添加header）。

curl也可以做到并发，基本流程如下
```php
$multiHandler = curl_multi_init(); //返回curl批处理句柄
foreach($requests as $name=>$request){
    $conn[$name] = curl_init($url);   //初始化新的会话，返回 cURL 句柄
    curl_setopt($conn[$name], CURLOPT_HTTPHEADER, $headers);//根据第二个参数为curl句柄设置不同的参数选项
    curl_multi_add_handle($multiHandler, $conn[$name]); //将句柄添加到批处理会话
    do {
        $mrc = curl_multi_exec($multiHandler, $active); //处理批处理句柄中的每一个连接
        usleep(10 * 1000);
        } while ($active);  
    foreach ($conn as $name => $con) {
        $result = json_decode(curl_multi_getcontent($con), true);  //获取句柄的输出
        if(empty($result) || empty($result["data"])) {
            continue;
        }
        $res[$name] = $result["data"];
        curl_close($con);   //关闭会话，释放资源
    }
}
```

其中的curl_multi_exec在句柄没有完全执行完毕时（$active>0）会不断循环执行，这样会导致cpu占用率很高，本例中使用usleep暂停请求，但时间长短不好把控，太短没有意义，太长会加大接口请求耗时。比较好的办法是使用curl_multi_select()，改写批处理句柄发送请求的方式，
```php
do {
    $mrc = curl_multi_exec($multiHandler, $active);
} while ($mrc == CURLM_CALL_MULTI_PERFORM);               //用$active判断会一直占用cpu

while ($active && $mrc == CURLM_OK) {               //active>0批处理句柄中还有待处理的句柄，并且上一次的读取或者写入句柄已经完毕
    if (curl_multi_select($multiHandler) != -1) {         //有活动链接才执行，不会一直占用cpu
        do {
            $mrc = curl_multi_exec($multiHandler, $active);
        } while ($mrc == CURLM_CALL_MULTI_PERFORM);
    }
}
```

其中的CURLM_CALL_MULTI_PERFORM表示数据还在读取或者写入当中，CURLM_OK表示上一次句柄的读取或者写入已经完成，核心是curl_multi_select在有活动连接的时候返回非-1，进入do-while循环发送请求，否则执行空循环，避免长时间占用cpu。

yar并行调用的缺陷，一个是无法单独添加header，一个是请求时需要重新请求回本机，并行curl请求克服了yar这两个缺陷，因此性能高于yar，测试也表明前者耗时更少。