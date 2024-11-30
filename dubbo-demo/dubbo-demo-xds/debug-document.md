# 01 环境配置
## 1.1 安装Docker Desktop
前往 **Docker** 官网下载安装。[https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/)


安装完成后，在 **Docker Desktop**中点击 **设置**-> **kubernetes**-> **Enable kubernetes**开启k8s集群。

> 注意： Mac 开启 k8s 集群时可能会存在拉取镜像问题，解决方法可参考 [https://blog.csdn.net/qq_43705697/article/details/143894239](https://blog.csdn.net/qq_43705697/article/details/143894239)
>

## 1.2 安装istio
下载对应的 istioctl 安装包 [https://github.com/istio/istio/releases](https://github.com/istio/istio/releases)

进入到下载包所在路径，执行命令`istioctl install`进行安装。

![](https://cdn.nlark.com/yuque/0/2024/png/12647363/1732375873262-71c4641d-ba91-457a-9b3f-747743a66e90.png)



> **注意：若 Mac电脑 安装过程中提示无法校验安全性，此时先不要关闭弹出窗口，只需要打开 「设置」-「隐私与安全性」-「仍要运行」，随后再执行一次`istioctl install` 命令，就会看到一个弹窗，点击打开，即可安装。
>

# 02 远程K8s调试示例
## 2.1 开启镜像仓库
部署示例时会在本地打包并推送镜像，所以需要先在本地启动一个镜像仓库。

执行如下命令后，会自动在本地启动一个镜像仓库容器用于存放镜像。

```shell
docker run -d -p 5000:5000 --restart=always --name local-registry registry:2
```

## 2.2 拉取&编译代码
**1、执行命令拉取Dubbo的`feature/xds`分支**

```shell
git clone -b feature/xds https://github.com/apache/dubbo.git
```

**2、代码格式化**

```shell
mvn spotless:apply
```

**3、编译代码时跳过测试**

```shell
mvn clean install -DskipTests
```

## 2.3 运行示例
在`dubbo/dubbo-demo/dubbo-demo-xds`目录下执行`./start.sh`命令即可运行示例。

`start.sh`脚本主要完成的任务如下：

1、新建名为`dubbo-demo`的`namespace`，并切换到此`namespace`。**

**2、构建`dubbo-demo-xds-provider`和`dubbo-demo-xds-consumer`镜像，并推送至刚刚开启的本地镜像仓库。构建镜像时将`resource/bootstrap.json`文件拷贝至镜像 `/bootstrap.json`目录下，同时开启远程`debug`端口。**

**3、通过`service.yaml`文件，创建`k8s`资源。**

**4、端口转发，将`istiod`的`15010`端口进行转发，方便本地直连`istiod`。将d`ubbo-demo-xds-consumer`服务的`31000`端口进行转发，方便远程`debug`。**

## **2.4 IDEA开启远程debug**
运行`start.sh`脚本后，通过`Docker Desktop`查看对应`pod`日志，可以看到`dubbo-demo-xds-provider`服务会自动运行，而`dubbo-demo-xds-**consumer`服务暂时挂起，等待调试中。此时需要编辑本地`idea调试配置`，增加断点，即可开始调试。

1、编辑调试配置

![](https://cdn.nlark.com/yuque/0/2024/png/12647363/1732984798806-18f77ef3-7f8f-4a10-a03c-721b5f8002fc.png)

2、新增`Remote JVM Debug`类型的配置，端口设置为`31000`，`module`选择`dubbo-demo-xds-consumer`。

![](https://cdn.nlark.com/yuque/0/2024/png/12647363/1732984890317-ff1bb796-010b-4ccb-896d-e7ecfdf44343.png)

**3、新增断点后，点击调试按钮，即可进行远程调试。**

---

**特别说明**

1、`dubbo-demo-xds-consumer`服务挂起的原因是因为通过`service.yaml`文件部署资源时设置了`suspen=y`，如果仅仅是运行示例，不需要调试，可以修改为`suspend=n`，编译代码后，重新执行`start.sh`进行部署，此时会看到两个服务都会启动。

![](https://cdn.nlark.com/yuque/0/2024/png/12647363/1732985564723-ff3e9d9b-189b-45a5-9e49-6ba4c20076e5.png)

2、对于开发人员，每次修改`dubbo-xds`模块代码后，都需要重新执行`mvn spotless:apply`代码格式化，然后执行`mvn clean install -DskipTests`编译打包，最后执行`./start.sh`构建镜像，重新部署容器。

