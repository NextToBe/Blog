#GET 与 POST 的区别
之前我们提到get与post的区别，无非提到以下几点：

* GET在浏览器回退时是无害的，而POST会再次提交请求
* GET产生的URL地址可以被Bookmark，而POST不可以
* GET请求会被浏览器主动cache，而POST不会，除非手动设置
* GET请求只能进行URL编码，而POST支持多种编码方式。
* GET请求参数会被完整保留在浏览器历史记录里，而POST中的参数不会被保留
* GET请求在URL中传送的参数是有长度限制的，而POST没有。
* 对参数的数据类型，GET只接受ASCII字符，而POST没有限制
* GET比POST更不安全，因为参数直接暴露在URL上，所以不能用来传递敏感信息
* GET参数通过URL传递，POST放在Request body中


再继续深入一点：可能会涉及到HTTP的幂等性(idempotent)，但也可能仅限于：

*  HTTP GET方法用于获取资源，不应有副作用，所以是幂等的。比如：GET <http://www.bank.com/account/123456>，不会改变资源的状态，不论调用一次还是N次都没有副作用。请注意，这里强调的是一次和N次具有相同的副作用，而不是每次GET的结果相同。GET <http://www.news.com/latest-news> 这个HTTP请求可能会每次得到不同的结果，但它本身并没有产生任何副作用，因而是满足幂等性的
*   POST所对应的URI并非创建的资源本身，而是资源的接收者。比如：POST <http://www.forum.com/articles> 的语义是在 <http://www.forum.com/articles> 下创建一篇帖子，HTTP响应中应包含帖子的创建状态以及帖子的URI。两次相同的POST请求会在服务器端创建两份资源，它们具有不同的URI；所以，POST方法不具备幂等性。而PUT所对应的URI是要创建或更新的资源本身。比如：PUT <http://www.forum/articles/4231> 的语义是创建或更新ID为4231的帖子。对同一URI进行多次PUT的副作用和一次PUT是相同的；因此，PUT方法具有幂等性。

但是，他们真的是这样吗？在这两天的开发中，测试用例里面，GET请求的body带上请求参数是完全可行的，*因为不管是POST还是GET在底层都是TCP/IP协议，那么他们便都是TCP连接，所以在GET请求的body里加上参数或者在POST请求的的URI里加上请求参数是完全可以的*。**因为HTTP协议并没有规定这些内容，但是各家浏览器和各家的服务器做了一些限制**。如果浏览器拒绝发起带有body的GET请求或者服务器拒绝解析GET请求的body，那么即使理论和技术上这样的方法都可以行得通，那么也是不实际的。

*所以，GET和POST本质上就是TCP连接，并无差别。但是由于HTTP的规定和浏览器/服务器的限制，导致他们在应用过程中体现出一些不同。*

但是，区别的并不仅限于此，还有更重要的区别：
> GET产生一个TCP数据包；POST产生两个TCP数据包。

对于GET方式的请求，浏览器(以 chrome 为例)会把http header和data一并发送出去，服务器响应200（返回数据）；

而对于POST，浏览器先发送header，服务器响应100 continue，浏览器再发送data，服务器响应200 ok（返回数据）。