# 一.测试环境

## 1.1 操作系统

- 中标麒麟 V7.0

## 1.2 测试工具

- Postman
- JMeter

## 1.3 测试方法

测试方法通过 z-ledger-sdk-go 编写相关 http 接口服务，该接口服务提供了一些常用的 Z-Ledger 操作，例如创建通道，加入通道，安装链码和调用链码等。然后使用 Postman 发送 http 请求到本地接口服务，再由本地服务调用本地运行的 Z-Ledger 网络进行相关操作。

在进行测试前，先启动本地的 Z-Ledger 网络，默认启动一个 Orderer 服务和两个 Peer 节点服务，启动网络操作可参考《Z-Ledger使用手册》。

1. 获取 z-ledger-sdk-go 源码：

```bash
$ cd $GOPATH/src/github.com/zhigui
$ git clone https://github.com/zhigui-projects/z-ledger-sdk-go.git
# 切换到v1.0.0-beta1版本
$ git checkout v1.0.0-beta1
```

2. 获取 http 接口服务源码：

```bash
$ cd $GOPATH/src/github.com/zhigui
$ git https://github.com/maluning/z-ledger-sdk-go-sample.git
```

下载完源码后，需要把z-ledger-sdk-go-sample/config 目录下的 org1-config.yaml 和 org2-config.yaml 中有关crypto-config的路径替换成本地路径，其他配置也可根据本地 Z-Ledger网络进行修改。

3. 启动 http 接口服务:

```bash
$ cd $GOPATH/src/github.com/hyperledger/z-ledger-sdk-go-sample
$ go build
# 启动z-ledger网络服务
$ ./z-ledger-sdk-go-sample z-ledger
# 开启另一个终端
# 启动ca服务器
$ ./z-ledger-sdk-go-sample ca
```

接下来就可以打开 Postman 通过访问 http 接口服务进行测试了。

# 二.功能测试

## 2.1 链码操作

### 2.1.1 安装链码

在 Postaman 地址栏中输入localhost:9090/resmgmt/install/v1进行安装链码，v1表示的是当前链码的版本。

![image-20200618163015470](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618163015470.png)

### 2.1.2 实例化链码

对上一步安装的链码进行实例化操作，该操作完成后链码中记录的a的值被初始化为100，b的值被初始化为200。

![image-20200618163245585](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618163245585.png)

### 2.1.3 查询链码

对安装的链码进行查询操作，验证链码是否被成功实例化。从以下返回结果看a和b已经被成功初始化。

![image-20200618163742924](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618163742924.png)

![image-20200618163803268](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618163803268.png)

### 2.1.4 调用链码

链码中的invoke函数实现的功能是将a向b转账10。

![image-20200618164205855](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164205855.png)

转账成功，a的余额为90，b的余额为210。

![image-20200618164312708](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164312708.png)

![image-20200618164333811](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164333811.png)

### 2.1.5 删除数据

链码中的delete函数定义了删除某个账户的操作，这里我们删除a账户。

![image-20200618164508290](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164508290.png)

操作成功后，我们再次查询a时，返回结果为空，即说明a已经被删除。

![image-20200618164623035](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164623035.png)

## 2.2 通道操作

### 2.2.1 创建通道

创建名businesschannel的通道。

![image-20200618164816140](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164816140.png)

### 2.2.2 加入通道

加入刚才创建好的通道。

![image-20200618164945157](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618164945157.png)

## 2.3 账本操作

### 2.3.1 注册账本事件

通过以下方式来注册账本事件。

![image-20200618165118951](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618165118951.png)

### 2.3.2 注册链码事件

通过以下方式注册链码事件,其中URL中最后一个参数为交易ID。

![image-20200618165405444](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618165405444.png)

### 2.3.3 注册交易状态事件

通过以下方式注册交易状态事件。

![image-20200618165544822](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618165544822.png)



### 2.3.4 注销注册事件

通过以下方式注销掉之前注册的事件。

![image-20200618165651024](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618165651024.png)

## 2.4 账本操作

### 2.4.1 查询账本信息

通过以下方式查询账本信息，可以看到返回结果中包含了区块高度为8，当前区块的哈希值和前一个区块的哈希值等。

![image-20200618165811953](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618165811953.png)

### 2.4.2 查询区块信息

通过以下方式查询第二区块的信息，可以看到返回了区块的相关信息和区块中携带的数据。

![image-20200618170018023](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618170018023.png)

### 2.4.3 查询交易信息

通过以下方式查询特定交易ID的相关信息，将返回该交易编码后的内容数据信息。

![image-20200618170330348](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200618170330348.png)



## 2.5 CA服务

### 2.5.1 注册证书

向CA服务注册证书需要两个步骤，分别是register和enroll。首先需要登陆一个操作用户（引导用户），该用户为admin，密码为adminpwd。

![image-20200707155634793](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200707155634793.png)



用以下方式为名为user的用户注册证书。先进行register，其中URL中的user表示用户名，userpw为用户密码。

![image-20200706174314610](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200706174314610.png)

接着进行enroll。

![image-20200706174511161](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200706174511161.png)

查看本地/tmp/examplestore目录我们将看到之前生成的证书：

```bash
$ ls
admin@Org1MSP-cert.pem	user@Org1MSP-cert.pem
```



# 三.性能测试

使用 JMeter 可对可能存在的大量频繁的请求接口（例如查询/调用链码，查询交易等）进行并发压力测试，下面采用在单机部署Z-Ledger网络的环境下进行测试。

## 3.1 链码查询并发测试

查询链码我们设置1000个并发线程模拟1000个用户，这1000个线程同时查询a的余额。JMeter相关参数设置如下：

![image-20200621220713216](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621220713216.png)

![image-20200621220805616](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621220805616.png)

启动程序后，查看测试结果报告，我们可以看到测试全部通过，总查询时间为5s，平均查询时间是3237ms，从图形结果可以看到，吞吐量达到了10917/分钟，且随着请求的增加，响应时间也是线性增加，这说明在单机部署情况下还是能很好的满足用户响应要求。

![image-20200621220623562](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621220623562.png)

![image-20200621220435773](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621220435773.png)



## 3.2 链码调用并发测试

链码调用功能为a向b转账10元，由于调用涉及Orderer的执行，存储数据的修改，为避免冲突导致事物回滚，设置发送请求数为100，每隔5s发送一个调用请求，测试结果如下。所有调用均能正确返回，拥有良好的准确性，平均响应时间为2084ms

![image-20200621230354007](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621230354007.png)

![image-20200621230640343](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621230640343.png)

## 3.3 交易查询并发测试

交易查询设置1s内发送并发线程数为1000，测试结果如下，可以看到测试用例全部通过，平均响应值为5111ms,吞吐量也达到了6749/分钟。

![image-20200621221342290](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621221342290.png)

![image-20200621221410195](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621221410195.png)

## 3.4 账本查询并发测试

账本查询设置1s内发送并发线程数为1000，测试结果如下，平均响应时间为3329ms，吞吐量为10491/分钟。

![image-20200621223019054](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621223019054.png)

![image-20200621223048759](/Users/jjvincent/Library/Application Support/typora-user-images/image-20200621223048759.png)



# 四.测试结论

通过在中标麒麟操作系统上进行Z-Ledger系统的功能和性能测试，从测试数据来看，功能测试均能返回正常结果，响应时间也符合一般用户对响应的要求。性能测试设置了较大的并发数来模拟多用户同时发送请求的情况，对于分布式系统来说，响应结果和响应时间均符合行业标准要求。

