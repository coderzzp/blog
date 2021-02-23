---
title: "服务端缓存策略之node实践"
date: 2021-02-23T14:02:36+08:00
draft: false
---

## 浏览器缓存策略之node实战
浏览器缓存是web开发中减少服务器压力相当常见也是相当有效的一种方式，那么本文会总结浏览器缓存的主要策略，并且最后会给出一个带缓存的node服务器代码实践。
### 浏览器缓存策略讲解
以下讲解全部基于服务器端支持缓存策略，若不支持，则每次都会想服务端发起请求，即不存在缓存
第一次访问某个网站的时候：
![image](https://user-images.githubusercontent.com/24691802/42408493-04ada050-8200-11e8-8558-bbbba00a6702.png)
浏览器会缓存这个响应的所有信息

之后再次访问这个网站的时候：会判断是否命中强缓存（根据expiress和cache-control），如果命中强缓存，会直接读取之前缓存过的资源，如果未命中强缓存，那么执行弱缓存策略，回头看一下我们之前缓存过的请求头中，Etag和Last-modified两个字段，我们的浏览器会将这两个缓存过的头放在请求头中，但是这个时候Etag变成了If-none-match，Last-modified变成了If-modified-since，服务器收到请求后会根据这两个字段判断请求的资源是否新鲜（即是否有改动），如果新鲜，返回304，如果不新鲜，则获取新资源，返回200.
![image](https://user-images.githubusercontent.com/24691802/42408508-4b7b12a6-8200-11e8-8285-868c613f30d7.png)
浏览器把缓存类型根据是否需要向服务器发起请求分为两种：**强缓存**和**弱缓存（协商缓存）**    

 缓存类型 |获取资源方式| 状态码| 是否发送请求到服务器|
--------- | --------- | --------|---------|
强缓存  |从缓存取 | 200(from disk cache) | 否，直接从缓存取|
弱缓存  | 从缓存去取 |304(not modified)| 是，通过服务器来告诉浏览器缓存是否可用 |  
------------

### 强缓存详解 
* expires，这是http1.0时的规范；它的值为一个绝对时间的GMT格式的时间字符串，如Mon, 10 Jun 2015 21:31:12 GMT，如果发送请求的时间在expires之前，那么本地缓存始终有效，否则就会发送请求到服务器来获取资源
* cache-control：max-age=number，这是http1.1时出现的header信息，主要是利用该字段的max-age值来进行判断，它是一个相对值；资源第一次的请求时间和Cache-Control设定的有效期，计算出一个资源过期时间，再拿这个过期时间跟当前的请求时间比较，如果请求时间在过期时间之前，就能命中缓存，否则就不行；cache-control除了该字段外，还有下面几个比较常用的设置值：
  * no-cache：不使用本地缓存。需要使用缓存协商，先与服务器确认返回的响应是否被更改，如果之前的响应中存在ETag，那么请求的时候会与服务端验证，如果资源未被更改，则可以避免重新下载。
  * no-store：直接禁止游览器缓存数据，每次用户请求该资源，都会向服务器发送一个请求，每次都会下载完整的资源。
  * public：可以被所有的用户缓存，包括终端用户和CDN等中间代理服务器。
  * private：只能被终端用户的浏览器缓存，不允许CDN等中继缓存服务器对其缓存。        

           ps：如果cache-control与expires同时存在的话，cache-control的优先级高于expires
### 弱缓存详解
弱缓存都是由服务器来确定缓存资源是否可用的，所以客户端与服务器端要通过某种标识来进行通信，从而让服务器判断请求资源是否可以缓存访问，这主要涉及到下面两组header字段，这两组搭档都是成对出现的，即第一次请求的响应头带上某个字段（Last-Modified或者Etag），则后续请求则会带上对应的请求字段（If-Modified-Since或者If-None-Match），若响应头没有Last-Modified或者Etag字段，则请求头也不会有对应的字段。
* Last-Modified/If-Modified-Since
  * 浏览器第一次跟服务器请求一个资源，服务器在返回这个资源的同时，在respone的header加上Last-Modified的header，这个header表示这个资源在服务器上的最后修改时间
  * 浏览器再次跟服务器请求这个资源时，在request的header上加上If-Modified-Since的header，这个header的值就是上一次请求时返回的Last-Modified的值
  * 服务器再次收到资源请求时，根据浏览器传过来If-Modified-Since和资源在服务器上的最后修改时间判断资源是否有变化，如果没有变化则返回304 Not Modified，但是不会返回资源内容；如果有变化，就正常返回资源内容。当服务器返回304 Not Modified的响应时，response header中不会再添加Last-Modified的header，因为既然资源没有变化，那么Last-Modified也就不会改变，这是服务器返回304时的response header
  * 浏览器收到304的响应后，就会从缓存中加载资源
  * 如果协商缓存没有命中，浏览器直接从服务器加载资源时，Last-Modified的Header在重新加载的时候会被更新，下次请求时，If-Modified-Since会启用上次返回的Last-Modified值
* Etag/If-None-Match：这两个值是由服务器生成的每个资源的唯一标识字符串，只要资源有变化就这个值就会改变；其判断过程与Last-Modified/If-Modified-Since类似，与Last-Modified不一样的是，当服务器返回304 Not Modified的响应时，由于ETag重新生成过，response header中还会把这个ETag返回，即使这个ETag跟之前的没有变化。
### Etag相对Last-modified的区别
* 一些文件也许会周期性的更改，但是他的内容并不改变(仅仅改变的修改时间)，这个时候我们并不希望客户端认为这个文件被修改了，而重新GET；
* 某些文件修改非常频繁，比如在秒以下的时间内进行修改，(比方说1s内修改了N次)，If-Modified-Since能 检查到的粒度是s级的，这种修改无法判断(或者说UNIX记录MTIME只能精确到秒)；    

        ps：浏览器会优先验证Etag，在一致的情况下，才会继续判断Last-modified。
### 实战
浏览器缓存主要是需要服务端去做处理，那我们这里大致介绍一下服务端的请求头设置，以及弱缓存判断是否新鲜的方式：
```javascript
setFreshHeaders(stat, res) {
      //关于缓存可以在https://www.cnblogs.com/wonyun/p/5524617.html参考
      //从stat里拿到信息
      const lastModified = stat.mtime.toUTCString();
      //强缓存
      if (this.enableExpires) {
          const expireTime = (new Date(Date.now() + this.maxAge * 1000)).toUTCString();
          res.setHeader('Expires', expireTime);
      }
      if (this.enableCacheControl) {
          res.setHeader('Cache-Control', `public, max-age=${this.maxAge}`);
      }
      //弱缓存
      if (this.enableLastModified) {
          res.setHeader('Last-Modified', lastModified);
      }
      if (this.enableETag) {
          res.setHeader('ETag', this.generateETag(stat));
      }
    }
```
这里的stat是可以通过fs.stat()这个api取到资源文件的相关信息，res是传入的response对象。
关于Etag的组成方式，通过查询mdn得到了这样的答案
![image](https://user-images.githubusercontent.com/24691802/42412445-07da69fe-823f-11e8-84fc-9acf9b61f15a.png)
我们这里使用的是最后修改事件戳的哈希值，实际上用文件内容的哈希会更加准确（原因参见etag和last-modified区别第一条）
```javascript
//设置etag的一种简单方式
    generateETag(stat) {
      const mtime = stat.mtime.getTime().toString(16);
      const size = stat.size.toString(16);
      return `W/"${size}-${mtime}"`;
    }
```
那么当服务器受到带有if-modified-since和if-none-match的请求时，会去判断资源是否新鲜，判断代码如下：
```javascript
isFresh(reqHeaders, resHeaders) {
      const  noneMatch = reqHeaders['if-none-match'];
      const  lastModified = reqHeaders['if-modified-since'];
      if (!(noneMatch || lastModified)) return false;
      if(noneMatch && (noneMatch !== resHeaders['etag'])) return false;
      if(lastModified && lastModified !== resHeaders['last-modified']) return false;
      return true;
    }
```
可以参考[完整代码](https://github.com/coderzzp/how2-learn-nodejs/tree/master/node-static-server)，测试一下刚才所说过的强弱缓存的策略，同时也学习一下用node如何写一个带缓存的静态服务器       

        我们在测试页面强缓存时不要用F5刷新，这样并不会命中强缓存，建议新开或者使用前后键来刷新页面 


 [参考问题](https://webmasters.stackexchange.com/questions/25342/headers-to-prevent-304-if-modified-since-head-requests)


