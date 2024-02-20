### HTTP 状态码

状态码 | 状态 | 含义
 :----: | :---- | :----
100 | Continue | 初始的请求已经接受，客户应当继续发送请求的其余部分
101 | Switching Protocols | 服务器将遵从客户的请求转换到另外一种协议
200 | OK | 一切正常，对 GET 和 POST 请求的应答文档跟在后面。
201 | Created | 服务器已经创建了文档，Location 头给出了它的 URL。
202 | Accepted | 已经接受请求，但处理尚未完成。
203 | Non-Authoritative Information | 文档已经正常地返回，但一些应答头可能不正确，因为使用的是文档的拷贝
204 | No Content | 没有新文档，浏览器应该继续显示原来的文档。如果用户定期地刷新页面，而 Servlet 可以确定用户文档足够新，这个状态代码是很有用的
205 | Reset Content | 没有新的内容，但浏览器应该重置它所显示的内容。用来强制浏览器清除表单输入内容
206 | Partial Content | 客户发送了一个带有 Range 头的 GET 请求，服务器完成了它
300 | Multiple Choices | 客户请求的文档可以在多个位置找到，这些位置已经在返回的文档内列出。如果服务器要提出优先选择，则应该在 Location 应答头指明。
301 | Moved Permanently | 客户请求的文档在其他地方，新的 URL 在 Location 头中给出，浏览器应该自动地访问新的 URL。
302 | Found | 类似于 301，但新的 URL 应该被视为临时性的替代，而不是永久性的。
303 | See Other | 类似于 301/302，不同之处在于，如果原来的请求是 POST，Location 头指定的重定向目标文档应该通过 GET 提取
304 | Not Modified | 客户端有缓冲的文档并发出了一个条件性的请求（一般是提供 If-Modified-Since 头表示客户只想比指定日期更新的文档）。服务器告诉客户，原来缓冲的文档还可以继续使用。
305 | Use Proxy | 客户请求的文档应该通过 Location 头所指明的代理服务器提取
307 | Temporary Redirect | 和 302 相同。许多浏览器会错误地响应 302 应答进行重定向，即使原来的请求是 POST，即使它实际上只能在 POST 请求的应答是 303 时才能重定向。由于这个原因，HTTP 1.1 新增了 307，以便更加清除地区分几个状态代码：当出现 303 应答时，浏览器可以跟随重定向的 GET 和 POST 请求；如果是 307 应答，则浏览器只能跟随对 GET 请求的重定向。
400 | Bad Request | 请求出现语法错误。
401 | Unauthorized | 客户试图未经授权访问受密码保护的页面。应答中会包含一个 WWW-Authenticate 头，浏览器据此显示用户名字 / 密码对话框，然后在填写合适的 Authorization 头后再次发出请求。
403 | Forbidden | 资源不可用。
404 | Not Found | 无法找到指定位置的资源
405 | Method Not Allowed | 请求方法（GET、POST、HEAD、Delete、PUT、TRACE 等）对指定的资源不适用。
406 | Not Acceptable | 指定的资源已经找到，但它的 MIME 类型和客户在 Accpet 头中所指定的不兼容
407 | Proxy Authentication Required | 类似于 401，表示客户必须先经过代理服务器的授权。
408 | Request Timeout | 在服务器许可的等待时间内，客户一直没有发出任何请求。客户可以在以后重复同一请求。
409 | Conflict | 通常和 PUT 请求有关。由于请求和资源的当前状态相冲突，因此请求不能成功。
410 | Gone | 所请求的文档已经不再可用，而且服务器不知道应该重定向到哪一个地址。它和 404 的不同在于，返回 407 表示文档永久地离开了指定的位置，而 404 表示由于未知的原因文档不可用。
411 | Length Required | 服务器不能处理请求，除非客户发送一个 Content-Length 头
412 | Precondition Failed | 请求头中指定的一些前提条件失败
413 | Request Entity Too Large | 目标文档的大小超过服务器当前愿意处理的大小。如果服务器认为自己能够稍后再处理该请求，则应该提供一个 Retry-After 头
414 | Request URI Too Long | URI 太长
416 | Requested Range Not Satisfiable | 服务器不能满足客户在请求中指定的 Range 头
500 | Internal Server Error | 服务器遇到了意料不到的情况，不能完成客户的请求
501 | Not Implemented | 服务器不支持实现请求所需要的功能。例如，客户发出了一个服务器不支持的 PUT 请求
502 | Bad Gateway | 服务器作为网关或者代理时，为了完成请求访问下一个服务器，但该服务器返回了非法的应答
503 | Service Unavailable | 服务器由于维护或者负载过重未能应答。例如，Servlet 可能在数据库连接池已满的情况下返回 503。服务器返回 503 时可以提供一个 Retry-After 头
504 | Gateway Timeout | 由作为代理或网关的服务器使用，表示不能及时地从远程服务器获得应答
505 | HTTP Version Not Supported | 服务器不支持请求中所指明的 HTTP 版本


### 服务端口号
服务与端口号的映射关系excel文件：[service_port](../files/others/service-names-port-numbers.xlsx)

https://www.iana.org/assignments/service-names-port-numbers/service-names-port-numbers.xhtml


