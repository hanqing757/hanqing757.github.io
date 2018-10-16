---
layout: post
title:  "生成真正的随机数"
date:   2018-10-16 20:01:00
categories: true-ramdom
tags: true-ramdom
excerpt: 依据random.org提供的API生成真正的随机数
mathjax: true
---

* content
{:toc}

通常我们在计算机上使用编程语言生成的随机数是按照一定的规则算法生成的，不具有真正意义上的随机性，属于伪随机数。[random.org](https://www.random.org/)提供了一组API接口通过HTTP协议来获取随机数，它的随机性来自于大气噪声，随机性优于一般编程语言中的生成的随机数。[trueRandom](https://github.com/hanqing757/trueRandom)基于[random.org](https://www.random.org/)提供的接口进行封装产生各类随机数，本文主要记录封装过程中遇到的两点问题。

[trueRandom](https://github.com/hanqing757/trueRandom)使用guzzlehttp这个库发送HTTP请求，guzzlehttp发送请求的方式，
```php
use GuzzleHttp\Client;
$client = new Client([
    'base_uri' => 'http://httpbin.org',
    'timeout'  => 2.0,
]);
$response = $client->request('GET', 'test'); //http://httpbin.org/test
$response = $client->request('GET', '/root'); //http://httpbin.org/root
//或者
$response = $client->get('http://httpbin.org/get');
$response = $client->post('http://httpbin.org/post');
//再或者创建一个Request
use GuzzleHttp\Psr7\Request;
$request = new Request('POST', 'http://httpbin.org/post');
$response = $client->send($request, ['timeout' => 2]);
```
当发送post请求时，通常可以使用'json'(application/json)上传json数据，'form_params'(application/x-www-form-urlencoded)上传键值对，'multipart'(multipart/form-data)上传form表单或者文件。
```php
$response = $client->request('POST', '/post', ['json' => ['foo' => 'bar']]);
```
其中，json对应的value是一个Array。
```php
$client->request('POST', '/post', [
    'form_params' => [
        'foo' => 'bar',
        'baz' => ['hi', 'there!']
    ]
]);
```
上传文件使用
```php
$client->request('POST', '/post', [
    'multipart' => [
        [
            'name'     => 'foo',
            'contents' => 'data',
            'headers'  => ['X-Baz' => 'bar']
        ],
        [
            'name'     => 'baz',
            'contents' => fopen('/path/to/file', 'r')
        ],
        [
            'name'     => 'qux',
            'contents' => fopen('/path/to/file', 'r'),
            'filename' => 'custom_filename.txt'
        ],
    ]
]);
```
也可以使用'body'来控制请求的主体部分，body是一个原生字符串。不过在GuzzleHttp6.3中好像已废弃。GuzzleHttp同样可以设置请求发送头，cookie等。
对于响应部分
```php
$code = $response->getStatusCode(); // 获取状态码
echo $response->getHeader('Content-Length'); //获取响应头
$body = $response->getBody();   //获取消息主体
//通过(string)$body或者$body->getContents()获取响应的string表示形式
```

再将string经json_decode转换为array。
那么对于json_encode和serialize，json_encode(\$value) 对变量进行json编码，value可以是除了 source 之外的所有类型，一般对array进行编码，当对object进行编码时，只保留public的实例属性，public的static以及protected和private的属性不保留，方法也不保留。
serialize(\$value)产生一个可存储的值表示，返回字符串，包含$value的字节流，有利于传递或者存储php的值，同时不丢失其类型和结构。当对object进行序列化时可以保留public的非static属性，protected和private属性，static属性和方法不能保留。
```php
class testJsonEncode{
	public $a=10;
	public static $aa=9;
	protected $b=11;
	private $c=12;
	public function getA(){
		echo $this->a;
	}
} 
$x=new testJsonEncode();
echo json_encode($x);     //{"a":10}
echo serialize($x);  //O:14:"testJsonEncode":3:{s:1:"a";i:10;s:4:"*b";i:11;s:17:"testJsonEncodec";i:12;}
```
当使用json_decode(string,true)的第二个参数设置为true时返回array而非对象，默认是false。