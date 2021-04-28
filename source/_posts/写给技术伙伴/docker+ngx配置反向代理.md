---
title: docker+ngx配置反向代理
---

![20210428145359](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C43053eff4f116c20dfc84312c0a85bb7.png)

<!-- toc -->
## 写在前面

这篇文章写给一个姓港的同学，希望他能学会基本的ngx使用，然后再去实现自己的想法

## 准备容器    
启动 `ngx` 并关联 80 端口
``` bash
docker run -d -p 80:80 --name nginx nginx:latest
```

进入容器，这个控制台留着刷新 `ngx` 配置使用
``` bash
docker exec -it nginx bash
```

## 修改配置文件
![20210428132825](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C3daa2ffd517177b6239fc8d7a3923d27.png)

查询 [docker ngx](https://hub.docker.com/_/nginx) 的文档可知，配置文件在 `/etc/nginx/conf.d/default.conf`, 新增一个控制台，将docker里面的ngx配置文件拷贝出来进行调整
```
docker cp nginx:/etc/nginx/conf.d/default.conf ./default.conf
```

进行ngx的配置修改，然后将文件拷贝回去(下面这段配置是简单地一个代理)
```
server {
    listen       80;
    listen  [::]:80;
    server_name  localhost;

    location /{
        proxy_pass http://enbrands-2.oss-cn-shanghai.aliyuncs.com/;
    }
}
```

## 重新加载 `ngx` 并进行预览
进行拷贝
``` bash
docker cp  ./default.conf nginx:/etc/nginx/conf.d/default.conf
```

在容器里面执行命令
``` bash

# 可以通过命令查看修改情况
more /etc/nginx/conf.d/default.conf

# 重启ngx
nginx -s reload
```

可以通过 http://localhost/user/d3a1a93e2d572937d7708b55d660bf46.png 查看效果

可以测试一下下列两个图片的展示情况
- http://enbrands-2.oss-cn-shanghai.aliyuncs.com/user/d3a1a93e2d572937d7708b55d660bf46.png
- http://localhost/user/d3a1a93e2d572937d7708b55d660bf46.png

![20210428134735](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C80f4f37f0ca9616b694ab0c911fe2a19.png)


- 如果要修改端口映射，启动阶段的时候 -p 8080:80， 则需要用 localhost:8080 进行访问