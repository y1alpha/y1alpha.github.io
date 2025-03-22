---
date: 2024-08-21 18:55
---

项目中对外提供的接口都是 https 的，但是在用户调用的时候可能会出现使用 http 调用的情况，所以需要将 http 自动跳转为 https 以增加用户体验。
查看资料，在 Nginx 中的配置文件 `nginx.conf` 中增加如下配置：
```
server {
    listen 80 default_server;
    server_name _;
    return 301 https://$host$request_uri;
}
```
在使用 postman 自测时发现，当请求接口为 post 的时候会出现 404 错误
![16757771161326](media/16757771161326.jpg)
此时查看 api 网关日志发现，该接口的请求方式变成了 get 请求
![16757773736174](media/16757773736174.jpg)
经过排查发现虽然 http 的标准要求浏览器在收到该响应并进行重定向时不应该修改 http method 和 body，但是有一些浏览器可能会有问题。最好是在应对 GET 或 HEAD 方法时使用 301。针对需要重定向 post 请求时需要使用 307，当发生重定向请求时 307 可以确保请求方法和消息主体不会发生变化。

具体的一些 http 响应码说明可以参考这篇文档 https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status

将配置改成如下后，post 接口就可以正常请求了。
```
server {
    listen 80 default_server;
    server_name _;
    return 307 https://$host$request_uri;
}
```
