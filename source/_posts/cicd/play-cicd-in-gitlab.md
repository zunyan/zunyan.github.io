---
title: 在gitlab中玩转cicd
date: 2021-03-28
---

<!-- toc -->

## 什么是CI/CD

持续集成（CI）和持续部署（CD）是DevOps中重要的两个环节，传统的软件开发和交付方式在迅速变得过时。过去的敏捷时代里，大多数公司的软件发布周期是每月、每季度甚至每年，而在现在 DevOps 时代，每周、每天甚至每天多次都是常态。

开发团队通过软件交付流水线（Pipeline）实现自动化，以缩短交付周期，并通过自动化流程来检查代码并部署到新环境。以快速的进行敏捷迭代和开发。

在实际使用中，gitlab的CI/CD是通过开发者预先配置的一系列`pipeline`参数，通过精心配置的触发时机，在代码提交、合并、或者打tag时，触发自动构建流来完成构建到发布的动作。

## 任务目标
![20210330104247](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C78a3acf6a1fcebbc87e43b7e22893fbe.png)

我们的任务目标是搭建一个 gitlab + gitlab-runner 的CICD环境，在代码触发时，启动构建动作，构建完毕后将代码推送到应用服务器上进行部署

应用服务是是一个具备nginx的服务，他暴露了80端口允许你访问端口，应用我们选择前端的 `hexo` 博客系统，开箱即用
## 准备
    
为了实现本文档的目标任务，需要做一下软件的前期准备

- 安装docker
- 了解docker常用操作
- apt 软件安装操作
## 相关材料
1. [docker安装gitlab](https://docs.gitlab.com/omnibus/docker/sunsky2017)
2. [docker四种网络模式，容器localhost访问宿主机端口](https://blog.csdn.net/ma726518972/article/details/108146218)
3. [linux 安装 nodejs](https://www.runoob.com/nodejs/nodejs-install-setup.html)
5. [linux软连接](https://www.runoob.com/linux/linux-comm-ln.html)
6. [安装 gilab-runner](https://docs.gitlab.com/runner/install/docker.html)
7. [Job artifacts](https://docs.gitlab.com/ee/ci/pipelines/job_artifacts.html)
7. [Job artifacts](https://docs.gitlab.com/ee/ci/pipelines/job_artifacts.html)

## 环境准备

### 从安装一个 gitlab 开始

首先我们在docker安装一个gitlab用来做本次实验

``` bash
# 下载镜像
docker pull gitlab/gitlab-ee

# 启动
docker run -d \
  --hostname localhost \
  -p 80:80 \
  --name gitlab \
  gitlab/gitlab-ee:latest

# 查看日志
docker logs -f gitlab
```

启动阶段要做较多的初始化工作，需要耐心等待。完成后可以通过 `8080` 端口看到gilab。

![20210330151758](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5Cbf329e38d119b379cb3f789d6e2fb9fd.png)


### 安装并注册 git runner

安装好 gitlab 后还要安装 gitlan-runner 并注册，gitlab-runner 主要用于 响应 gitlab CI/CD，CI/CD里面的script脚本将会被 gitlab-runner 所执行

``` bash
docker pull gitlab/gitlab-runner
docker run -d \
    --name gitlab-runner \
    --network host \  # 共享主机网络保证gitrunner能够正确的访问gitlab
    gitlab/gitlab-runner:latest

# 命令模式进入容器
docker exec -it gitlab-runner bash
```

进入容器控制台，输入如下命令进行注册
```
gitlab-ci-multi-runner register
```

注册时需要一个token参数，可以访问 http://localhost/admin/runners 这个页面去获取

![20210328171907](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C3d7d9be41d8d383572bee29ae8fe2827.png)


``` bash
> gitlab-ci-multi-runner register

Runtime platform arch=amd64 os=linux pid=63 revision=943fc252 version=13.7.0
Running in system-mode.

Enter the GitLab instance URL (for example, https://gitlab.com/):
> http://localhost/

Enter the registration token:
> Q1L_vwDETgx8Fx-Yypfp

Enter a description for the runner:
[23229cabbaf2]: demo
Enter tags for the runner (comma-separated):

Registering runner... succeeded                     runner=Q1L_vwDE
Enter an executor: docker-ssh, virtualbox, kubernetes, docker-ssh+machine, custom, docker, parallels, shell, ssh, docker+machine:
> shell
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```

值得注意的是URL填写的时候，不能使用 https://localhost/ 请使用本机IP, 配置完后页面刷新后你会看到一个新注册的 runner

![20210330163620](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C8f309188cc584c17efa3acd6a7494fdf.png)

然后点击编辑，将 lock 那一项给点掉

![20210330163658](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C799f2e5bd172ae56f0a24985939d6731.png)

最后返回列表你会看到

![20210330161142](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C5d3b019e51bbd20a3090c252df4afe87.png)


为了能运行nodejs项目，还需要继续安装 `nodejs`
``` bash

apt update

# 安装解压工具
apt install xz-utils bzip2 -y

# 下载
wget https://nodejs.org/dist/v10.9.0/node-v10.9.0-linux-x64.tar.xz  

# 解压wget 
tar -xf node-v10.9.0-linux-x64.tar.xz

# 建立软连接，让npm和node可以在全局调用
ln -b -s /node-v10.9.0-linux-x64/bin/npm /bin/npm
ln -b -s /node-v10.9.0-linux-x64/bin/node /bin/node

# 查看node版本
node -v
```

- exector 表示执行器，表示执行脚本时，使用的命令行程序，配置docker比较复杂，此处直接使用shell执行器

### 安装一台用于部署的服务器

至此，我们还需要一台用于部署应用的服务器，由于需要使用ssh进行连接（gitlab-runner使用该端口做远程部署），我们使用 `nginx` 镜像， 并在上面安装一个 `openssh-server`，最后打开ssh通道，并配置账号密码允许ssh访问

首先，先pull nginx 镜像, 然后启动nginx

``` bash
docker pull nginx
docker run -d --name nginx \
    -p 8080:80 \
    nginx:latest 
```

启动完毕后，8080端口就可以直接访问了 http://localhost:8080/

## 搭建代码仓库和cicd
### 新建一个仓库

至此，runner的执行环境基本做完了，接下来我们需要新建一个代码仓库，然后配置CI/CD的相关内容。

我们需要先创建一个仓库

![20210330141010](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5Cf2ab9131e8bb0829a987ca75ac5a03e7.png)

找到

![20210330141031](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C1973f0595af1c9c8b28ca46068f41296.png)

新建

![20210330141102](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C653cb725b258eb3dd604f18bddef8768.png)
![20210330141103](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C653cb725b258eb3dd604f18bddef8768.png)

## 测试CI/CD

模板默认已经配好了CICD，可以直接运行

### 配置cicd

仓库建完后，在仓库目录下面有一个 `.gitlab-ci.yml` 文件，点开编辑


``` yml
image: node:10.15.3

cache:
  paths:
    - node_modules/

before_script:
  - test -e package.json && npm install

pages:
  script:
    - ./node_modules/hexo/bin/hexo generate
    # scp需要进行进行ssh相关的配置才可以使用，未完成
    # - scp -r public root@localhost:/usr/share/nginx/html
  artifacts:
    paths:
      - public
  only:
    - master

```

保存完毕后，CI/CD就开始执行

![20210331134531](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5Cc93bb4359ac1d2dfce5a0c8f450189ea.png)

### 发布程序

cicd执行完毕后，由于我们配置了 `artifacts` 参数，可以在ci/cd面板中下载构建产物
![20210331172338](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C9fc048eb2effa8757aa2dc7a4c29dd9d.png)

我们可以直接在应用服务器上面下载这个产物 [细节看此链接](https://docs.gitlab.com/ee/ci/pipelines/job_artifacts.html#access-the-latest-job-artifacts-by-url)

``` bash
docker exec -it nginx bash
# 到ngx的目录
cd /usr/share/nginx/
# 下载最后一个在master上面构建的包
curl --output artifacts.zip --header "PRIVATE-TOKEN: s8Y9J1g5Azof89zfhEhN" "http://192.168.0.157/api/v4/projects/root%2Fblog/jobs/artifacts/master/download?job=pages" 

```
命令中的 Ip 请修改成自己的IP，不要使用localhost, `root%2Fblog` 是项目路径 `root/blog` encode之后的，`PRIVATE-TOKEN` 需要在仓库的 `Setting -> access token` 获得
![20210331174424](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5C01553d39ef2a554bd9889b169925a4ae.png)


下载完毕后可以使用 `ls` 查看
``` bash
root@6aca3f15f8d8:/usr/share/nginx# ls
50x.html  artifacts.zip  index.html
```

然后我们将其解压
``` bash

apt update # 更新软件源
apt install unzip # 安装解压软件

unzip artifacts.zip

rm -rf html # 删除旧文件
mv public html # 将目录移到ngx配置目录
```

![20210331180401](https://zunyan.oss-cn-hongkong.aliyuncs.com//images%5Cblog%5Cfbba1ce4a37b3ddc0e1b7d1930573e2e.png)

就可以看到结果了（样式问题是工程自己的问题）

### 使用`SCP`直接在cicd中发布
todo
