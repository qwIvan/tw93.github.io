---
layout:     post
title:      高性能JavaScript--Ajax
date:       2015-02-14 21:47:29
summary:    Ajax是高性能JavaScript的基础，它可以通过延迟下载体积较大的资源文件来使页面加载更快，它通过异步的方式在客户端和服务器之间传递数据，从而避免页面资源一窝蜂的下载。它甚至可以只用一个HTTP请求就获得整个页面的资源。选择合适的传输方式和最有效的数据格式，可以显著改善用户和网站的交互体验。
categories: JavaScript  学习笔记
---

Ajax是高性能JavaScript的基础，它可以通过延迟下载体积较大的资源文件来使页面加载更快，它通过异步的方式在客户端和服务器之间传递数据，从而避免页面资源一窝蜂的下载。它甚至可以只用一个HTTP请求就获得整个页面的资源。选择合适的传输方式和最有效的数据格式，可以显著改善用户和网站的交互体验。我们这里主要讨论从服务器接收发数据速度最快的技术，以及最为有效的数据编码格式。  

###常用的向服务器请求数据的技术  

#### XMLHttpRequest(XML)
使用范例：
{% highlight javascript %}
var url = '/data.php';
var params = [
    'id=9798',
    'limit=20'
];
var req = new XMLHttpRequest();
req.onreadystatechange = function() {
    if (req.readyState === 4) {
        //获取响应的头信息
        var responseHeaders = req.getAllResponseHeaders();
        //获取数据
        var data = req.responseText;
        //数据处理

    }
}
req.open('GET', url + '?' + params.join('&'), true);
//设置请求头信息
req.setRequestHeader('X-Request-With', 'XMLHttpRequest')；
req.send(null); //发送一个请求
{%endhighlight%}

当使用XHR请求数据时候，对于那些不会改变服务器状态，只会获取数据的请求，应该使用GET。经GET请求的数据会被缓存起来，如何需要多次请求统一数据，它会有助于提高性能。
只有当请求URL加上参数长度接近或者超过2048个字符时候，才应该使用POST获取数据，这是因为**IE限制了URL长度**，过长请求会被截断。

####动态脚本注入

这种方法克服了XHR的最大限制：它能跨域请求。你不需要实例化一个专用对象，而可以使用JavaScript创建一个新的脚本标签，并设置它的属性为不同域的URL,但是和XHR相比，动态脚本注入提供的控制是有限的，你不能设置请求头信息，传递参数也只能使用GET，不能设置请求的超时处理和重试，也就是说失败了你也不一定知道。你必须等到所有数据都返回你才可以访问它们。你不能请求头信息，也不能把整个响应信息作为字符串来处理。还有必须是可执行的JavaScript源码，而且必须封装在一个回调函数中。
{% highlight javascript %}
var scriptElement=document.createElement('script');
scriptElement.src='http://any-domain.com/javascript/lib.js';
document.getElementByTagName('head')[0].appendChild(scriptElement);

function jsonCallback(jsonString){
    var data=eval('('+jsonString+')');
    //处理数据...
    }
{% endhighlight %}
在上面那个例子里，lib.js必须把数据封装在jsonCallback函数里面：
{% highlight javascript %}
jsonCallback({"status":1,"color":["#fff","#000","f00"]});
//尽管有很多限制，但是这项技术的速度非常快。
{%endhighlight%}

####Multipart XHR
MXHR允许客户端只使用一个HTTP请求就可以从服务端向客户端传送多个资源，它通过在服务端将资源（CSS文件，HTML片段，JavaScript代码，或base64编码的图片）打包成一个双方约定的字符串并发送到客户端，然后用JavaScript代码处理这个长字符串，并根据它的mime-type类型和传入其他头信息解析出每个资源。
但是以这种技术获得的资源不能够被浏览器缓存 ，但是某些情况下MXHR依然能显著提高页面的整体性能。

###发送数据给服务器

####XMLHttpRequest
数据可以使用GET或POST的方式传回来，包括任意数量的HTTP头信息，当使用XHR发送数据给服务器时候，使用GET会更快，只需要发送一个数据包，POST至少两个数据包，一个装载头信息，一个装载POST正文。POST更适合发送大量数据到服务器。
{% highlight javascript %}
var url = '/data.php';
var params = [
    'id=9798',
    'limit=20'
];
var req = new XMLHttpRequest();
req.onerror=function(){
    //出错
    };
req.onreadystatechange = function() {
    if (req.readyState === 4) {
        //成功
    }
}
req.open('POST',url, true);
req.setRequstHeader('Content-Type','application/x-www-form-urlencoded');
req.setRequstHeader('Content-Length',params.length);
req.send(params.join('&'));
{%endhighlight%}

####Beacons（图片信标）
这项技术非常类似动态脚本注入。使用JavaScript创建一个新的Image对象，并把src属性设置为服务器上传脚本的URL。该URL包含我们通过GET传回的键值对数据。请注意并没有创建img元素或把它插入DOM。
{% highlight javascript%}
var url='/status_tracker.php';
var params=[
'step=2',
'time=123241223'
];
(new Image()).src=url+'?'+params.join('&');
{% endhighlight %}

服务器会接收数据并保存下来，它无需向客户端发送任何回馈信息，因为没有图片会实际显示出来，这是往服务器回传信息最有效的方式。它的性能消耗很小，而且服务器的错误完全不会影响客户端。如果你需要返回大量数据给客户端，那么请使用XHR，如果你只关心发送数据给服务器（可能需要极少的返回信息），那么请使用图片信标。

###数据格式的选择，         
通常来说数据格式约轻量越好，JSON和字符串分割的自定义格式是最好的。如果数据集很大并且对解析时间有要求，那么请从如下两种格式中做出选择。
 - JSON-P格式，使用动态脚本注入获取他把数据当作可以执行的JavaScript而不是字符串，解析速度极快。它能够跨域使用，但涉及敏感数据的时候不应该使用它。
 - 字符分割的自定义格式，使用XHR或动态脚本注入获取，用split()解析。这种技术解析大数据集比JSON-P略快，而且通常文件尺寸更小。

###Ajax性能指南
一旦选择了最合适和数据传输技术和数据格式，那么你就得开始考虑其他优化技术了：
 - 减少请求数，可以通过合并JavaScript和CSS文件，或者使用MXHR。
 - 缩小页面的加载时间，页面主要内容加载完成后，用Ajax获取那些次要的文件。
 - 确保你的代码错误不会输出给用户，并在服务端处理错误。
 - 知道何时使用成熟的Ajax类库，以及编写自己的底层Ajax代码。