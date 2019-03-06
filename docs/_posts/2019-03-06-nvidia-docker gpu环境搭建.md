docker gpu环境搭建
=============

前言
-------------
搭建GPU的开发环境需要安装nvidia的驱动、cuda、cudnn等，还要安装tensorflow、pytorch、mxnet等框架，并且驱动版本和框架版本需要相统一，如tensorflow1.9的版本需要对用cuda9.0，如果要升级tensorflow，cuda也要做相应的升级。每次在新机器上部署环境都费时费力，因此急需一套docker来快速移植。


安装nvidia-docker
-------------
普通的docker环境不支持gpu，因此我们需要一个[nvidia-docker的版本](https://github.com/NVIDIA/nvidia-docker)。如图：
![](/blog/images/nvidia-docker.png)
我们可以下载官方的带驱动的镜像，这样我们就只需要在容器中安装cuda toolkit就行了。

* 首先我们需要选择安装nvidia-docker的版本，查询可用版本：
```
apt-cache madison nvidia-docker2 nvidia-container-runtime
```

* 在列表中我选择18.09.2的版本进行安装
nvidia-docker2和nvidia-container-runtime的版本最好相同
```
sudo apt-get install -y nvidia-docker2=2.0.3+docker18.09.2-1  nvidia-container-runtime=2.0.0+docker18.09.2-1
```

* 装好后测试驱动
nvidia的官方镜像有[很多](https://hub.docker.com/r/nvidia/cuda/)，比如我使用cuda-9.0的版本，这样测试：
```
sudo nvidia-docker run --runtime=nvidia --rm nvidia/cuda:9.0-base nvidia-smi
```

* 在官方镜像上装其他cuda-toolkit和其他环境
我利用有驱动的官方镜像安装其他环境，首先把docker跑起来
```
sudo nvidia-docker run -it  nvidia/cuda:9.0-base
```

接下来，完善此镜像。安装完整的cuda-toolkit：
官方的镜像不完整，这一步很关键
```
apt install cuda-toolkit-9-0
```

* 然后根据自己的需要安装其他程序，比如我安装了python、pip、tensorflow、pytorch等。

* 安装好自己的环境后，退出，保存：
commit后面是ps出来的容器ID，后面跟自己想要保存的名字，我这里是chaos_gpu
```
sudo nvidia-docker ps -a
sudo nvidia-docker commit adaf25712ec8 chaos_gpu
```

* 注册仓库上传很麻烦，我这里直接保存成文件
```
docker save -o chaos_gpu_image chaos_gpu
```

* 最后可以测试自己的程序了
比如我的一个服务是需要开端口监听，docker里面监听80端口，把它映射到宿主机的8080端口上去
宿主机就可以使用http://127.0.0.1:8080/进行访问了
```
sudo nvidia-docker run -it -p 8080:80  chaos_gpu  your_bash
```
