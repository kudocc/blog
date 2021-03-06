# Use one NSURLSession

我们项目中使用了AFNetworking，然后做了一层封装，对于每个请求都新建了一个`AFHTTPSessionManager`，一直觉得这样挺不好的，不过因为并不是我负责的部分就没有在意。我第一次意识到这个有问题是在读AFNetworking的源码的时候，我发现了内存泄漏（内存泄露是使用不当造成的，并非AFNetworking的问题），因为其内部组合了`NSURLSession`，而`NSURLSession`的delegate是retain的，而这个delegate就是`AFHTTPSessionManager`实例，所以`AFHTTPSessionManager`是有循环引用的。不过如果你的项目中不是向我们一样每个请求都新创建一个`AFHTTPSessionManager`，而是只用一个单例的话，那么这并没有问题。

还有一个问题稍微高级一点，其实我主要想说这个问题，哈哈。我们工程基本上就是一个base url，照理说可以使用HTTP的persistent connections特性的，即第一个请求会建立tcp连接，在请求之后连接不会立即断开，之后发向同一个地址的请求还是会是用这个连接，省去了TCP的3次握手。不过我们工程中每次请求都创建了一个新的NSURLSession对象，我有点怀疑是不是每次发送请求都会重新建立tcp？我就尝试着抓了一下包，果然，每次发送请求都要进行tcp的三次握手，浪费流量、延迟又高。于是我修改代码，让每次请求重用同一个`NSURLSession`，再次抓包，发现之后的请求不会重新建立tcp连接了。Done！

所以对于同一个域名发送的请求，要尽量使用一个NSURLSession实例，HTTP默认使用头`Connection: keep-alive`，这样就可以尽量使多个请求使用同一条tcp连接，保证了低延迟。不过，即使这样也未必会保证两个请求使用同一条tcp连接，这也依赖于服务端http服务器的设置，就我所知，apache可以设置keep-alive的超时时间和同一条连接最多可以被多少个请求使用。在服务端response headers中可能会看到：`Keep-Alive: timeout=15, max=100`。这里我参考了[这个](http://stackoverflow.com/questions/19155201/http-keep-alive-timeout)和[这个](http://stackoverflow.com/questions/12095533/proper-use-of-keepalive-in-apache-htaccess)。由于自己对服务端不十分了解，可能有错误的地方，忘指正。
