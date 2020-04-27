---
title: Jenkins+GitLab实现持续集成
abbrlink: 47535
date: 2020-04-27 01:11:48
---

## 环境
阿里云 Centos7 双核8G，并且这是在Docker上进行的
## 安装

```shell
yum -y install docker # 安装docker
systemctl start docker # 启动docker
docker search jenkins	# 搜索
docker pull jenkins/jenkins:lts	# 安装镜像（建议到官网找最新的版本，否则可能出现其他问题）
```
## 运行
```shell
# 在当前用户下创建文件夹jenkins
mkdir jenkins
chmod -R 777 jenkins
# 运行(docker指令自行百度)
docker run -itd -p 8091:8080 -p 8092:50000 --name jenkins --privileged=true -v ~/jenkins/:/var/jenkins_home jenkins
# 查看运行状态
docker ps
# 运行结果
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                                             NAMES
54f487e8570b        jenkins               "/bin/tini -- /usr/l…"   7 minutes ago       Up 59 seconds       0.0.0.0:8091->8080/tcp, 0.0.0.0:8092->50000/tcp   jenkins
```

## 首次运行配置

1. 配置密码

首次进入系统时会显示此界面，密码在`jenkins/secrets/initialAdminPassword`文件里，当设置好密码后，此文件会被删除

![image-20200417220707821](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200417220707821.png)

```shell
cat ~/jenkins/secrets/initialAdminPassword	# 获取密码
```

2. 安装插件

![image-20200417221232314](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200417221232314.png)

此处选择安装推荐的插件即可

![image-20200417221401932](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200417221401932.png)

3. 设置用户与密码

![image-20200417225601957](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200417225601957.png)

设置完成后就基本配置完成了。以后使用当前设置的账户即可登陆。

## 配置环境与服务器


1. 配置JDK与Gradle

打开Manage Jenkins -> Global Tool Configuration，然后看到JDK和Gradle并点击Installtion并进行配置。在此处我选择了安装Sum的JDK8和Gradle 6.0，因为我的项目是使用Gradle进行打包的。我们也可以根据路径指定当前系统中JDK、Maven等。

![image-20200425002243025](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425002243025.png)

![image-20200425002257686](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425002257686.png)

2. 添加SSH服务器

进入Manage Jenkins -> System Configuration并找到Publish over SSH -> SSH Servers选项，点击Add并进行配置。这个服务器在这里是作为打包测试通过之后发布的服务器。

![image-20200425002832843](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425002832843.png)

填写完成后点击Test Configuration进行测试，看是否能够连接成功。

## 创建第一个项目

1. 点击主页面的New Item，填入名字并选择Freestyle project。

![image-20200425003208597](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425003208597.png)

2. 配置代码获取的方式，此处输入代码仓库的地址和账号等信息。同时还可以指定跟踪的分支，例如develop分支或者是master分支。

![image-20200425003607177](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425003607177.png)

3. 配置触发自动部署的条件，当达到一定的条件之后就可以触发Jenkins获取最新的代码进行打包、测试、部署等。因为我的GitLab服务器是能够访问到Jenkins服务器的，所以我选择在上传代码之后，通过Webhook触发部署。同时在此处点击Generate生成Secret token, 待会配置到GitLab服务器中。

   > Jenkin支持的几种触发方式：
   >
   > 1. 通过远程触发(Trigger builds remotely)
   > 2. 在其他项目构建完成后出发(build after other projects are built)
   > 3. 周期性地进行检查(build periodically)
   > 4. 通过webhook进行触发(build when a change is pushed to GitLab)
   > 5. 定时检查源码变更情况，如有变更则触发(poll SCM)

![image-20200425004218750](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004218750.png)

4. 配置打包的过程，可以通过执行Shell命令、调用Gradle脚本等进行打包。我这里使用的是Gradle脚本进行打包的。

![image-20200425004241365](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004241365.png)

![image-20200425004316870](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004316870.png)

5. 最后可以配置打包成后所执行的步骤，此处配置的是打包测试成功后通过SSH发布。所以选择Send build artifacts over SSH。在SSH Server中选择刚刚配置的SSH Server即可。同时在transfer配置源文件等。这样在Jenkins中就已经配置完成了。

![image-20200425004403298](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004403298.png)

> Transfer Set
>
> Source files: 指打包后需要发布的War包或者是Jar包
>
> Remove prefix: 在Source files中，倘若填写的是/test/xx.jar，如果传到SSH服务器上指定的目录下，会自动创建test文件夹，并且xx.jar文件在test文件夹下。移除前缀就可以不用创建test文件夹
>
> Remote directory: 登陆SSH后进入的目录
>
> Exec command: 传输完文件后需要执行的命令

![image-20200425004457290](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004457290.png)

6. 配置GitLab中的WebHook，进入项目的Setting -> Intergrations 界面，填入配置Build Trigger中显示的URL与Secret Token，同时在下面选择触发条件。这个触发条件就是在什么条件下触发WebHook。

![image-20200425004613435](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004613435.png)

![](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004730148.png)

填写完成后点击Add webhook即可出现下面显示的一个条目。

![image-20200425004838397](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004838397.png)

点击Test下拉按钮能够显示多个事件，这个可以模拟上传代码、合并代码事件。这样就能够测试我们是否在上传的时候触发Jenkins的打包功能。

![image-20200425004855352](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425004855352.png)

如果成功了，点击Edit按钮滑动到底部，会看到一些回调信息已经发送成功的记录。当然，还是直接查看Jenkins是否已经触发了打包过程。我在测试的过程中就尝试过了，这里已经出现200的状态码，但是并没有触发打包。就是因为我在Jenkins配置的时候设置了一些过滤规则，过滤掉了。

![image-20200425005010500](https://raw.githubusercontent.com/KitwahSin/KitwahSin.github.io/pictures/images/image-20200425005010500.png)

参考：

[jenkins各种触发方式介绍](https://blog.csdn.net/anzhuo5151/article/details/101786217?depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1&utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1)

