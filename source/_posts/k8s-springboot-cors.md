---
title: k8s下的spirngboot跨域问题
date: 2019-12-04 16:35:05
tags:
  - k8s
  - springboot
  - cors
  - ingress-nginx
---

## 很烦的跨域问题

在前后端分离的情况下，假如在springboot没配置cors的情况下，就会出现这样的问题。

```
Access to XMLHttpRequest at 'http://localhost:8080/api' 
from origin 'http://localhost:63342' has been blocked by CORS policy: 
Response to preflight request doesn't pass access control check: 
No 'Access-Control-Allow-Origin' header is present on the requested resource.
```

那很简单，只要重写一下`WebMvcConfigurer`下的`addCorsMappings`方法就可以了，*真就这么容易就好了，草*

```java
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")
                .allowCredentials(true);
    }
```

## 思考

根据以往的经验，一般请求链如下，通过浏览器然后通过nginx反向代理到springboot应用，按照从前来一直都没问题
```
web browser --> nginx --> springboot
```

但是我自己写了一个ajax在本地测试，发现是没有问题的，但是推送到测试服务器供前端测试的时候却不行

```javascript
$.ajax({
            method: "POST",
            url: "http://localhost:8080/api",
        }).then(function (value) {
            console.log(value);
        })
```

## allowedHeader配置

然后前端那边提醒说到有自定义的请求头，然后我再测试，发现我本地也不行了

```javascript
$.ajax({
            method: "POST",
            url: "http://localhost:8080/api",
            headers:{
                "token":"abc"
            }
        }).then(function (value) {
            console.log(value);
        })
```

然后再修改springboot里的配置，在通过上面的ajax测试，发现成了，然后高兴推送到测试服务器，结果我再测试发现依然不行
```java
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")
                .allowedHeaders("token")
                .allowCredentials(true);
    }
```

## nginx

然后因为往常nginx也没有配置，发现也没有问题，然后这次居然不行，然后把目光投向k8s的`nginx-ingress`

接着google了一番，发现`Ingress`需要配置几个`Annotation`  
最后  
```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo
  namespace: demo
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "PUT, GET, POST, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"
    nginx.ingress.kubernetes.io/cors-allow-headers: "token"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
spec:
  rules:
  - host: api.demo.com
    http:
      paths:
      - path: /
        backend:
          serviceName: demo-service
          servicePort: 80
```

最后通过上述操作，终于可以了
