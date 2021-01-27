---
title: 区块链开源实现fabric快速部署及CLI体验
tags:
  - fabric
  - hyperledger
  - 区块链
id: '530'
categories:
  - - 区块链
  - - 算法
date: 2018-05-22 09:50:48
---

本文描述fabric快速部署的步骤，及演示基于官方example02的智能合约进行CLI命令行体验。区块链涉及服务很多，且大量使用docker容器技术，所以请严格遵守以下步骤去部署，以减少各种问题的出现，方便我们先对联盟链有个大概的感觉。本文描述环境是centos7操作系统，请其他版本更正相关的安装工具（如ubuntu操作系统请把yum命令换成apt-get）。
<!-- more -->
# 1、搭建e2e\_cli环境快速部署fabric的11个步骤：

1、安装docker\_ce版。 如果已经安装了老版docker，请先卸载。

```
yum remove docker docker-common docker-selinux docker-engine
```

再来安装docker：

```
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
sudo yum-config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo yum-config-manager --enable docker-ce-edge
sudo yum-config-manager --enable docker-ce-test
sudo yum-config-manager --disable docker-ce-edge
sudo yum makecache fast
sudo yum install docker-ce
```

最后启动docker服务：

```
service docker start
chkconfig docker on
```

2、配置好docker加速器。 官方docker非常慢，请一定在阿里云等提供docker仓库加速器的网站注册好帐户(比如[https://cr.console.aliyun.com/#/accelerator](https://cr.console.aliyun.com/#/accelerator))，配置好加速器。 3、安装好pip。

```
yum install python-pip -y
```

4、用pip安装docker-compose。

```
pip install docker-compose
```

5、新建存放测试、部署代码的目录。

```
mkdir -p /opt/gopath/src/github.com/hyperledger
```

6、安装git。

```
yum install git -y
```

7、拉取fabric代码。

```
cd /opt/gopath/src/github.com/hyperledger
git clone https://github.com/hyperledger/fabric.git
```

**请切换到最新的1.1分支上。** 8、拉取docker镜像（时间较长）及一些可执行文件。

```
./opt/gopath/src/github.com/hyperledger/fabric/scripts/bootstrap.sh
```

9、安装go语言。

```
yum install golang -y
```

并通过在/etc/profile最后追加两行设置好工作目录：

```
export PATH=$PATH:/usr/local/go/bin
export GOPATH=/opt/gopath
```

最后执行下：

```
source /etc/profile
```

10、**修改一个阻塞执行的bug**。（注：fabric最新代码已修复该bug，若你拉取了最新代码，请忽略该步骤！） 修改/opt/gopath/src/github.com/hyperledger/fabric/examples/e2e\_cli/base/peer-base.yaml文件：

```
将：
- CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=e2ecli_default
修改为：
- CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=e2e_cli_default
```

11、启动服务。 进入/opt/gopath/src/github.com/hyperledger/fabric/examples/e2e\_cli目录，执行：

```
bash network_setup.sh up
```

如果最后出现下图则部署成功： ![](/2018/05/fabric_cliexe例子-3.jpg)

# 2、体验fabric系统功能

当服务在一台server上启动后，可以看到以下docker实例： ![](/2018/05/fabric_cliexe-docker实例-1.jpg) 可以看到，默认安装了：4个peer（2个是org1的，2个是org2的）节点、4节点构成的kafka集群、3节点构成的zookeeper集群、1个orderer节点。这是因为：fabric提供的共识机制，PBFT目前还未达到生产级别的应用，只能靠kafka+zookeeper实现PAXOS算法下的共识机制（不能有作恶结点）。一般zookeeper是3或者5个结点， fabric提供了SDK和CLI两种交互方式，本文不讨论SDK。 可以进入cli里执行peer命令。如果cli长时间未用退出后，可先启动cli：

```
docker start cli
```

再进入cli实例里：

```
docker exec -it cli bash
```

接着可执行peer命令，体验区块链的命令行使用方式。

## 2.1、 peer命令

peer命令含有五个子命令，如下：

```
peer chaincode [option] [flags]
peer channel   [option] [flags]
peer logging   [option] [flags]
peer node      [option] [flags]
peer version   [option] [flags]
```

每个子命令可使用--help查看详细帮助。 在fabric里，所有的交易必须通过智能合约才能操作，而chaincode链码就是智能合约。chaincode支持以下option操作：

*   package 智能合约需要打包后才能使用
*   install    智能合约必须安装后才能使用
*   instantiate   置初始状态。比如设系统一开始用户a有100元，用户b有200元
*   invoke  调用智能合约
*   query    查询状态
*   signpackage  包签名
*   upgrade    智能合约升级
*   list        显示智能合约

智能合约需要先install才能使用。

```
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
```

其中-n表示合约名字，-p指向合约文件目录路径，-v是版本号。

## 2.2 example02智能合约

每个智能合约实现Init和Invoke两个方法，其中前者用于初始化，后者是日常调用。

### 2.2.1 初始化

example02的Init方法接收4个参数(`包括帐户A，余额pA，帐户B，余额pB`)，比如：

```
peer chaincode instantiate -o orderer.example.com:7050 -C mychannel -n mycc -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR    ('Org1MSP.peer','Org2MSP.peer')"
```

其中，-C指向channel名字，-c则是初始构造json格式的消息，-P是背书策略，-o指定共识节点。这里置帐户a初始余额为100，帐户b初始余额为200。其代码实现如下：

```
func (t *SimpleChaincode) Init(stub shim.ChaincodeStubInterface) pb.Response {
fmt.Println("ex02 Init")
        //用args取得命令行的参数
_, args := stub.GetFunctionAndParameters()
var A, B string    // Entities
var Aval, Bval int // Asset holdings
var err error
      
        //简单的例子么，只接收4个参数，包括帐户A，余额pA，帐户B，余额pB
if len(args) != 4 {
return shim.Error("Incorrect number of arguments. Expecting 4")
}

// Initialize the chaincode
A = args[0]
Aval, err = strconv.Atoi(args[1])
if err != nil {
return shim.Error("Expecting integer value for asset holding")
}
B = args[2]
Bval, err = strconv.Atoi(args[3])
if err != nil {
return shim.Error("Expecting integer value for asset holding")
}
fmt.Printf("Aval = %d, Bval = %d\n", Aval, Bval)

//将A及其余额写入状态数据库
err = stub.PutState(A, []byte(strconv.Itoa(Aval)))
if err != nil {
return shim.Error(err.Error())
}

        //将B及其余额写入状态数据库
err = stub.PutState(B, []byte(strconv.Itoa(Bval)))
if err != nil {
return shim.Error(err.Error())
}

return shim.Success(nil)
}
```

### 2.2.2 查询余额

接着查询余额，例如查询a帐户的余额：

```
peer chaincode query -C mychannel -n mycc -c '{"Args":["query","a"]}'
```

这里需要说明，example02的Invoke共支持3种指令：

```
func (t *SimpleChaincode) Invoke(stub shim.ChaincodeStubInterface) pb.Response {
fmt.Println("ex02 Invoke")
function, args := stub.GetFunctionAndParameters()
if function == "invoke" {
// 从A向B帐户转账
return t.invoke(stub, args)
} else if function == "delete" {
// 从状态数据库中删除帐户
return t.delete(stub, args)
} else if function == "query" {
// 查询状态数据库中某帐户的余额
return t.query(stub, args)
}

return shim.Error("Invalid invoke function name. Expecting \"invoke\" \"delete\" \"query\"")
}
```

因此，上述查询我们会得出类似结果： ![](/2018/05/fabric-cli-example02-query-3.jpg) 为何结果格式是这样的呢？看下t.query的实现：

```
func (t *SimpleChaincode) query(stub shim.ChaincodeStubInterface, args []string) pb.Response {
var A string // Entities
var err error

if len(args) != 1 {
return shim.Error("Incorrect number of arguments. Expecting name of the person to query")
}

A = args[0]  //获取帐户的名字

// 从状态数据库中获取帐户的值
Avalbytes, err := stub.GetState(A)
if err != nil {
jsonResp := "{\"Error\":\"Failed to get state for " + A + "\"}"
return shim.Error(jsonResp)
}

if Avalbytes == nil {
jsonResp := "{\"Error\":\"Nil amount for " + A + "\"}"
return shim.Error(jsonResp)
}

jsonResp := "{\"Name\":\"" + A + "\",\"Amount\":\"" + string(Avalbytes) + "\"}"
        //这里是显示在命令行的格式：Query Response:90
fmt.Printf("Query Response:%s\n", jsonResp)
return shim.Success(Avalbytes)
}
```

### 2.2.3 转帐

比如由B向A转帐50：

```
peer chaincode invoke -o orderer.example.com:7050  --tls true --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem  -C mychannel -n mycc -c '{"Args":["invoke","b","a","50"]}'
```

成功后我们会在输屏结果中找到这一行：ESCC invoke result: version:1 response:<status:200 message:"OK" > 其实现为从A和B帐户下用GetStat方法从状态数据库取出余额，与query类似，接着加减相应的值50后，再调用PutStat方法写入状态数据库。 此时我们可以再次查询，可以获得正确结果。   小结：以上只适用于简单体验fabric的功能，对于智能合约、共识算法、世界状态等在接下来的文章中我们再分析。