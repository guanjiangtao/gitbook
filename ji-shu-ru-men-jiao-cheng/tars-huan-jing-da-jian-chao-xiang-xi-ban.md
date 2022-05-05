---
coverY: 0
---

# Tars环境搭建（超详细版）

### 简介

Tars是基于名字服务使用Tars协议的高性能RPC开发框架，同时配套一体化的服务治理平台，帮助个人或者企业快速的以微服务的方式构建自己稳定可靠的分布式应用。

Tars是将腾讯内部使用的微服务架构TAF（Total Application Framework）多年的实践成果总结而成的开源项目。Tars这个名字来自《星际穿越》电影中机器人Tars， 电影中Tars有着非常友好的交互方式，任何初次接触它的人都可以轻松的和它进行交流，同时能在外太空、外星等复杂地形上，超预期的高效率的完成托付的所有任务。 拥有着类似设计理念的Tars也是一个兼顾易用性、高性能、服务治理的框架，目的是让开发更简单，聚焦业务逻辑，让运营更高效，一切尽在掌握。

目前该框架在腾讯内部，有100多个业务、10多万台服务器上运行使用。

### 整体架构

![架构拓扑图](https://tarscloud.github.io/TarsDocs/assets/tars\_tuopu.png)

整体架构的拓扑图主要分为2个部分：服务节点与公共框架节点。

服务节点:

服务节点可以认为是服务所实际运行的一个具体的操作系统实例，可以是物理主机或者虚拟主机、云主机。随着服务的种类扩展和规模扩大，服务节点可能成千上万甚至数以十万计。 每台服务节点上均有一个Node服务节点和N(N>=0)个业务服务节点，Node服务节点会对业务服务节点进行统一管理，提供启停、发布、监控等功能，同时接收业务服务节点上报过来的心跳。

公共框架节点:

除了服务节点以外的服务，其他服务节点均归为一类。

公共框架节点，数量不定，为了自身的容错容灾，一般也要求在在多个机房的多个服务器上进行部署，具体的节点数量，与服务节点的规模有关，比如，如果某些服务需要打较多的日志，就需要部署更多的日志服务节点。

### 框架部署

tars框架十分的简单，只依赖mysql即可完成部署。tars官网提供了四种部署方案，开发者可以根据自己的技能点，进行部署。

* 源码编译部署
* Docker 部署
* K8s Docker 部署
* K8s 融合部署

我这里选择的是Docker部署的方案。

上文提到Tars框架部署依赖于MySQL，所以在进行Tars运行之前，我们需要准备好MySQL环境依赖。这里提供2条路：

1.本机服务器安装MySQL。

2.依赖于第三方服务，比如腾讯云的MySQL或者阿里云的MySQL。

假如没有购买第三方服务的数据库的话，则建议使用第一种方案。

* 注意 MySQL 的 IP 和 root 密码，后续构建中需要使用

```
docker run -d -p 3306:3306 \    --net=tars \    -e MYSQL_ROOT_PASSWORD="123456" \    --ip="172.25.0.2" \    -v /data/framework-mysql:/var/lib/mysql \    -v /etc/localtime:/etc/localtime \    --name=tars-mysql \    mysql:5.6
```

具体可以查看地址：[https://tarscloud.github.io/TarsDocs/installation/docker.html](https://tarscloud.github.io/TarsDocs/installation/docker.html)

当MySQL准备好之后，开始部署 tarscloud/framework 框架。

1.拉取镜像

```
docker pull tarscloud/framework:latest
```

2.运行Docker

```
# 挂载的/etc/localtime是用来设置容器时区的，若没有可以去掉# 3000端口为web程序端口# 3001端口为web授权相关服务端口(docker>=v2.4.7可以不暴露该端口)docker run -d \    --name=tars-framework \    --net=tars \    -e MYSQL_HOST="172.25.0.2" \    -e MYSQL_ROOT_PASSWORD="123456" \    -e MYSQL_USER=root \    -e MYSQL_PORT=3306 \    -e REBUILD=false \    -e INET=eth0 \    -e SLAVE=false \    --ip="172.25.0.3" \    -v /data/framework:/data/tars \    -v /etc/localtime:/etc/localtime \    -p 3000:3000 \    -p 3001:3001 \    tarscloud/framework:v3.0.4
```

安装完毕后, 访问 `http://${your_machine_ip}:3000` 打开 web 管理平台，出现如下界面，则代表初始化成功。

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2F8Eeyx9jvBBD3c6xo0d7F%2Fimage.png?alt=media\&token=ec703580-64e7-40ba-b60f-4beead505a8f)

### TarsGo开发

此处略过Go环境开发

#### 安装TarsGo工具

安装 `tarsgo` 创建项目脚手架:

```
# < go 1.17 go get -u github.com/TarsCloud/TarsGo/tars/tools/tarsgo# >= go 1.17go install github.com/TarsCloud/TarsGo/tars/tools/tarsgo@latest
```

安装 `tars2go` 工具:

```
# < go 1.17 go get -u github.com/TarsCloud/TarsGo/tars/tools/tars2go# >= go 1.17go install github.com/TarsCloud/TarsGo/tars/tools/tars2go@latest
```

#### 服务编写

创建服务

```
tarsgo make App Server Servant GoModuleName例如：tarsgo make TestApp HelloGo SayHello github.com/Tars/test
```

和其他微服务框架一样Tars也具备自己的pb文件，.tars结尾。接口文件定义请求方法以及参数字段类型等，有关接口定义文件说明参考 tars\_tup.md

为了测试，我们定义一下tars文件，WebApi.tars。

```
module ElasticSearchServiceApp{   struct Data   {      0 require string type;      1 require string value;   };​   interface WebApi   {      int Update(int id, string docType, vector<Data> data); // 更新接口      int Insert(vector<Data> data); // 插入数据      int Delete(int id, string docType); // 删除数据      int BulkInsert(map<string, Data> data); // 批量插入      vector<Data> Search(map<string, string> data); // 搜索   };};
```

服务端开发，我们先将tars协议文件转为Golang语言。

```
tars2go  -outdir=tars-protocol -module=ElasticSearchService WebApi.tars
```

编写接口

```
package main​import (   "ElasticSearchService/tars-protocol/ElasticSearchServiceApp"   "context"   "github.com/TarsCloud/TarsGo/tars")​// WebApiImp servant implementationtype WebApiImp struct {}​type ElasticSearchWebApiImp struct {}​func (imp *WebApiImp) Init() (ret int32, err error) {   return 1, nil}​func (imp *WebApiImp) Update(tarsCtx context.Context, id int32, docType string, data []ElasticSearchServiceApp.Data) (ret int32, err error) {​   return 0, nil}​func (imp *WebApiImp) Insert(tarsCtx context.Context, data []ElasticSearchServiceApp.Data) (ret int32, err error) {​   return 0, nil}​func (imp *WebApiImp) Delete(tarsCtx context.Context, id int32, docType string) (ret int32, err error) {   return 999, nil}​func (imp *WebApiImp) BulkInsert(tarsCtx context.Context, data map[string]ElasticSearchServiceApp.Data) (ret int32, err error) {​   return 0, nil}​func (imp *WebApiImp) Search(tarsCtx context.Context, data map[string]string) (ret []ElasticSearchServiceApp.Data, err error) {​   return []ElasticSearchServiceApp.Data{}, nil}
```

**注意**： 这里函数名要大写，Go 语言方法导出规定。

编译 main 函数，初始代码以及有 tars 框架实现了。

编译

```
make && make tar
```

生存ElasticSearchServiceApp可执行文件以及压缩包。

#### 验证服务

编写Client代码。

```
package main​import (   "ElasticSearchService/tars-protocol/ElasticSearchServiceApp"   "fmt"   "github.com/TarsCloud/TarsGo/tars")​func main() {   comm := tars.NewCommunicator()   obj := fmt.Sprintf("ElasticSearchServiceApp.ElasticSearchService.WebApiObj@tcp -h 127.0.0.1 -p 10015 -t 60000")   app := new(ElasticSearchServiceApp.WebApi)   comm.StringToProxy(obj, app)   ret, err := app.Delete(1, "1")   if err != nil {      fmt.Println(err)      return   }   fmt.Println(ret)}
```

输出结果验证

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2FKzpQcbYh0PzOtSFdQytr%2Fimage.png?alt=media\&token=8f471b2c-38c9-4744-911f-0e8fd615ab7a)

### TarsGo部署

接着上文提到，我们通过make tar生成了tgz包，此时我们只需要在界面上完成服务的创建，既可以发布。

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2FVFdYODA9CTmOa8mFuDII%2Fimage-20220506014112821.png?alt=media\&token=159d17af-ba45-4029-b2f6-43017e58681c)

部署完之后，我们回到服务管理模块。

选择刚添加的服务ElasticSearchService，选择发布管理，上传我们刚打出来的tgz包。

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2Fe0heNYkQvKQSPy5HOuTi%2Fimage.png?alt=media\&token=ceaef84d-4acb-40bf-be5a-c1f4a6c770fd)

进入发布管理，选择发布包进行上传，这样服务器上会有所有的发布版本，如果遇到问题也方便快速回滚。。

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2FiRajGyv3zpVHUUTGehdK%2Fimage.png?alt=media\&token=542c457f-0926-48de-b017-cfa95688ff6b)

上传完成，回到发布界面，点击发布，即可完成本次发布。

![](https://files.gitbook.com/v0/b/gitbook-x-prod.appspot.com/o/spaces%2F20kemn8RybZdnecgP6W6%2Fuploads%2Fxx622tz6Oj3rvVhI1Nuc%2Fimage.png?alt=media\&token=473b8bc7-0d01-4525-9384-6ef8d78ebaf9)

### 参考文献

Tars 官方文档 [https://tarscloud.github.io/TarsDocs/SUMMARY.html](https://tarscloud.github.io/TarsDocs/SUMMARY.html)

Tars 官网 [https://tarscloud.org/#](https://tarscloud.org/#)
