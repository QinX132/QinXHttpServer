# QinXComm
Secure communication and remote management

## 项目架构及功能说明

### 架构说明

QXComm分为几大模块： QXCommMngr、QXServer、QXClient和一些可以单独部署的小组件（redis/mongo）

QXCommMngr暴露在公网，提供以下服务：

1. 提供api接口，为QXClient提供当前负载最小的QXServer地址（可使用安全服务，为数据使用对应Client公钥加密传输）
2. 管理QXServer，包括：查看QXServer当前机器健康状态、查看当前QXServer当前注册在线的所有Client、使能/关闭QXServer对Client提供服务、为QXServer提供身份认证服务
3. 为QXClient、QXServer提供注册服务，生成密钥对，作为唯一身份信息（写入mongo）
4. 为QXClient提供互通域管理服务，只有在互通域之中的QXClient才能相互传递消息（写入mongo）

QXServer暴露在公网，提供以下服务：

1. 可以创建集群，多个QXServer作为负载均衡；或创建私有QXServer只为指定QXClient提供连接
2. 对外暴露接口，提供给QXClient提供连接，且只接受在QXCommMngr上注册过的QXClient的注册
3. 为互通域内的QXClient转发消息，拒绝非互通域内的消息转发
4. 向QXCommMngr提供所在机器的健康情况，以及QXServer的工作情况（写入redis）

QXClient可在公网可在局域网提供以下服务：

1. 注册后，并且添加互通域后，可以向指定QXClient发送消息
2. 对QXServer集群可以自动切换，在当前的QXServer连接出问题时，自动切换集群中其他备机

数据库出于安全考虑，部署在私网，提供服务：

1. mongo保存QXClient的公钥消息、互通域信息，在QXServer启动时，读取互通域消息
2. redis保存QXServer的相关健康信息

注：考虑到现在国家正在大力推进GmSSL，所以本项目除了SSL/TLS的安全服务都使用GmSSL算法库而非OpenSSL

### 功能说明

#### Ⅰ QXCommMngr权限控制

* 角色：manager

  仅manager可以注册互通域、获取QXServer管理信息

* 角色：partner

  partner（QXServer）可以在注册界面进行QXServer的注册

* 角色：user

  user（QXClient）可以访问注册界面进行QXClient的注册

#### Ⅱ QXServer注册

partner在QXCommMngr上登录后，可以注册QXServer，注册后会得到一个QXCommMngr签名的证书，作为身份证明

#### Ⅱ QXClient注册

QXClient在QXCommMngr上注册，并且获取一个id身份证明（公私钥对），公钥将在QXCommMngr的数据库留存

![注册流程](./PrjArchSrc/Register.png)

#### Ⅲ 互通域管理

manager角色可在前端界面设置互通域，指定某些Client可以相互通信。设置后，互通域保存在mongo数据库中，并且通知所有QXServer同步互通域，通过版本号管理。

![互通域管理](./PrjArchSrc/ManageIntraDomain.png)

#### Ⅳ QXClient消息交互

QXClient先通过QXCommMngr api，获取当前最佳的QXServer地址进行连接，并通过ssl/tls协议进行身份验证和信息加密。 连接后，同样通过公私钥对，在QXServer上进行注册。 注册完成后即可对互通域内的所有QXClient进行通信。

![消息交互](./PrjArchSrc/Communication.png)

#### Ⅴ QXServer监控以及远程控制、负载均衡

QXServer在启动后，定期向redis中写入监控数据例如cpu、内存使用情况，负载情况，由QXCommMngr进行远程管理，通过enable/disable来开启/关闭server转发服务。

![Server管理](./PrjArchSrc/ServerManager.png)

### 安全性说明

#### 1. QXCommMngr、QXServer之间的安全性

* QXCommMngr如何验证QXServer的合法性？

  * 在QXserver通过合法注册的partner注册后，会在QXCommMngr处生成一对SM2密钥对，并且生成一个独一无二的id

  * QXCommMngr留存公钥，以作为该id的身份标识
  * 在QXServer连接上QXCommMngr、建立SSL连接之后，要求其对发出的Enc(随机数，PubKey)进行解密，并且将解密出的随机数附上签名返回给QXCommMngr，若随机数一致以及验签成功，才会继续其他业务的处理

* QXServer如何验证QXCommMngr的合法性？

  * partner注册后，可以通过在QXServer预置QXCommMngr SSL/TLS证书，在建立SSL连接时指定该证书，以验证QXCommMngr

* 消息交互过程：

  * 使用SSL信源加密，保证安全性

  综上，QXCommMngr、QXServer之间的安全性，使用SM2算法、SSL/TLS协议提供身份认证、授权、加密服务，保证数据完整性、非否认性。

#### 2. QXCommMngr、QXClient之间的安全性

* QXClient如何验证QXCommMngr的合法性？

  * QXClient使用https服务，保证QXCommMngr的合法性

* QXCommMngr如何验证QXClient的合法性？

  * QXClient发起https请求时，必须附带id及对当前时间戳的SM2私钥加密

* 消息交互过程：

  * 仅一次交互，通过SM2算法保证安全性

  综上，QXCommMngr、QXClient之间的安全性，使用SM2算法、SSL/TLS协议提供身份认证、授权、加密服务，并且通过时间戳来防止重放攻击

#### 3. QXServer、QXClient之间的安全性

* QXClient、QXServer如何验证双方的合法性

  * 前置条件：QXServer在QXCommMngr注册后，会获取当前所有的QXClient的id和公钥（可以增量更新，防止每次数据量过大）。QXClient则是启动后，可以通过向QXCommMngr发起请求，获取当前QXServer的最佳可用地址及其公钥。

  * 注册过程：QXClient先发起注册请求，QXServer收到后，发出时间戳和随机数，通过QXClient公钥加密，QXClient解密后，将此二者返回，并且也生成自己的时间戳和随机数，使用QXServer公钥加密，发送给QXServer，要求其解密后返回。此过程验证完成后，QXCient、QXServer正式注册，再进行相关业务处理。

  * 消息转发过程：可选SM4预置密钥/协商密钥进行消息信源加密，保证转发过程中的安全性。
  * 而在QXClient-QXClient之间，则属于信源+信道加密

## 构建流程

本仓库使用cmake构建，适配类unix系统，若有某些系统不适配请联系作者。

版本信息：

cmake version 3.22.1 

gcc version 11.4.0 (Ubuntu 11.4.0-1ubuntu1~22.04) 

java version "17.0.6" 2023-01-17 LTS

Apache Maven 3.6.3

@vue/cli 5.0.8

### 1. 克隆代码仓

git clone --recursive  https://github.com/QinX132/QXComm.git (--recursive 克隆所有子模块的代码仓)

### 2. 构建

```sh
cd QXComm

./build -a
```

### 3. （选读）单独模块构建

#### Ⅰ 三方仓构建

在工程目录执行./build -t ，将会自动进入third_party，执行third_party_build_all.sh脚本，构建gmssl()、libevent库（优先构建静态库）

三方库依赖以及其版本：

> [submodule "third_party/GmSSL"]
>
> ​	path = third_party/GmSSL
>
> ​	url = https://github.com/guanzhi/GmSSL.git (commit:6de0e022)
>
> [submodule "third_party/json"]
>
> ​	path = third_party/json
>
> ​	url = https://github.com/nlohmann/json.git (commit:199dea11)
>
> [submodule "third_party/libevent"]
>
> ​	path = third_party/libevent
>
> ​	url = https://github.com/libevent/libevent.git (commit:4fd07f0e)


#### Ⅱ 部署protobuf
1. 下载protobuf并部署：

   ```sh
   wget https://github.com/protocolbuffers/protobuf/releases/download/v21.12/protobuf-all-21.12.zip
   unzip protobuf-all-21.12.zip
   cd protobuf-all-21.12
   ./configure
   make
   sudo make install
   sudo ldconfig                                      # refresh shared library cache.
   ```

#### Ⅲ 构建utils代码仓：

在工程目录执行./build -u ，将会自动进入utils，执行 utils_build.sh脚本，编译所有.c文件，并且将此代码仓的文件编译为.a文件。此代码仓包含了一些c语言编写的模块功能，包括安全管理模块、健康检查模块、日志模块、内存管理模块、命令行模块、线程池模块、定时器模块、网络消息模块、作者自己编写的双向循环链表，以及一些常用的api。（相关的模块说明待开发者补充）

该目录内置unittest，如果要执行单元测试，执行如下操作：

```sh
sudo apt-get install libcurl4-openssl-dev // 下载curl 4 openssl

cd utils

rm build -rf && mkdir build && pushd build

cmake -DBUILD_SHARED_LIBS=OFF -DBUILD_TESTS=ON ..

make && make test
```

#### Ⅳ 构建QXServer 、 QXClient 代码共享仓（主要是protobuf文件）

在工程目录执行./build -h ，将会自动进入SCShare，执行 scshare_build.sh脚本，编译.proto文件，并且将此代码仓的文件编译为.a文件

#### Ⅴ 构建 QXServer、QXClient 服务

在工程目录执行./build -sc ，将会自动进入QXServer、QXClient，执行脚本，编译。生成可执行文件QXServer/src/build/QXServer、 QXClient/src/build/QXClient。

## 模块说明

### util说明

### QXServer 说明

### QXClient 说明
