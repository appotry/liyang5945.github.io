---
title: 使用nginx修改接口响应头解决跨域问题

date: 2021-10-21 19:30

author: liyang

categories: 
    - 技术分享
toc: false
tags:
  - nginx
  - 跨域

---

跨域这个问题对于前端来说是老生常谈了，搜索一下相关问题就会出现一堆解决方案，有的方法是通过使用nginx反向代理来解决跨域问题，不过这种方是只存在两个域的情况下，比如说我们的前端页面发布在A服务器上，然后接口是B服务器提供，我们在A服务器上通过nginx反向代理请求B服务器的接口。不过我最近碰到的问题要比这个复杂点，现在我们有三个域，我们多了一个第三方的接口在C服务器上，这个接口是第三方提供，不受控制，而且因为一些原因配置了白名单，只能通过B服务器的ip来访问，这样我们只能把nginx放在B服务器上来转发C服务器的接口。

nginx配置如下

```
location /Api/ {
	     proxy_pass http://serverC/Api/;
       }
```

配置好了后，我们可以通过B服务器来访问C服务的接口了，但是浏览器却报了一个错：

![](https://images.liyangzone.com/article_img/技术相关/共用/net_err.png)

```
Access to XMLHttpRequest at 'http://XXX.com/API' from origin 'http://localhost:8080' has been blocked by CORS policy: The value of the 'Access-Control-Allow-Origin' header in the response must not be the wildcard '*' when the request's credentials mode is 'include'. The credentials mode of requests initiated by the XMLHttpRequest is controlled by the withCredentials attribute.
```
翻译过来就是：在localhost:8080向发起的XMLHttpRequest请求被CORS策略阻止了，当请求的凭证模式为'include'时，响应头'Access-Control-Allow-Origin'的值不能是通配符*，XMLHttpRequest 发起的请求的凭证模式由 withCredentials 属性控制


withCredentials这个属性你可以简单理解为跨域请求是否带cookie，默认是false，这里我们是需要cookie的所以必须设成ture，但是设置为ture的话，接口返回的响应头'Access-Control-Allow-Origin'的值就不能是*，必须是请求方的地址。在vue项目中，这个值一般在封装axios的代码里：

```
const service = axios.create({
  baseURL: '', // url = base url + request url
  withCredentials: true, // 配置跨域请求是否带cookies
  timeout: 300000 // request timeout
})

```


因为这个C服务器的接口不受控制，所以不能通过代码层面来解决，那么能不能使用nginx来修改这个响应头呢，答案是肯定的，查了一些资料并试验后，我成功解决了这个问题，首先我们通过配置proxy_hide_header把C接口的响应头Access-Control-Allow-Origin给去掉，然后再加上一个正确的响应头，来达到修改的效果，具体配置如下：

```
location /Api/ {
  # 配置OPTIONS请求，返回204状态  
  if ($request_method = 'OPTIONS') {
    add_header Access-Control-Allow-Credentials true;
    add_header Access-Control-Allow-Origin $http_origin;
    add_header Access-Control-Allow-Methods 'GET, POST, OPTIONS';
    add_header Access-Control-Allow-Headers 'DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization';
            return 204;
  }
  proxy_pass http://serverC/Api/;
  proxy_hide_header Access-Control-Allow-Credentials; # 隐藏错误响应头
  proxy_hide_header Access-Control-Allow-Origin; # 隐藏错误响应头
  add_header Access-Control-Allow-Credentials true; # 添加正确响应头
  add_header Access-Control-Allow-Origin $http_origin; # 添加正确响应头
}
```
配置中除了Access-Control-Allow-Origin还有个Access-Control-Allow-Credentials响应头，这个值也是必须的，而且必须为true，加上上面的配置之后就能愉快地开发了。


