---
title: 区块链开源实现hyperledger fabric架构详解
tags:
  - chaincode
  - channel
  - fabric
  - hyperledger
  - MSP
  - PKI
  - 公钥
  - 区块链
  - 比特币
  - 私钥
id: '543'
categories:
  - - 区块链
date: 2018-05-26 10:05:13
---

hyperledger fabric是区块链中联盟链的优秀实现，主要代码由IBM、Intel、各大银行等贡献，目前v1.1版的kafka共识方式可达到1000/s次的吞吐量。本文中我们依次讨论：区块链的共通特性、fabric核心概念、fabric的交易执行流程。本文来源于笔者欲对公司部分业务上链而进行培训的PPT，故图多文字少，不要怕太长。
<!-- more -->
# 1、区块链解决方案的特性

## 1.1 分布式帐本

区块链核心概念是分布式帐本，就像下面的图1所示，同样的帐本（全量的交易数据，详见下节）在任意一台节点（不包括客户端）上都有。所以，其优点是数据很难造假，造假后也可以通过追溯记录来追究法律责任。而缺点就是极大的浪费，传统服务每份数据都尽量少存几份，即使存了三份拷贝都已经考虑到诸多异常，并使服务可用性达到N个9了。而区块链这种特性，同时造成的另一个问题是帐本不能太大，至少不能超过区块链网络中最小结点的存储以及处理能力。所以，这制约了总交易数据（下文为方便概念介绍，统称为帐本ledger）的条数，进而也影响了能写入区块链的单条交易数据的大小。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/unnamed-file-3.png) 图1 区块链分布式帐本示意图 什么是区块链呢？我很喜欢《区块链技术进阶与实战》一书中对它的定义：区块链是一种按照时间顺序将数据区块以顺序相连的方式组合成的一种链式数据结构。如果觉得有点抽象，那么我们再来看看下面的图2。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/unnamed-file-4.jpg) 图2-区块链数据结构示意 图2中就是账本，它由多个区块构成了一个有时序的链表，而每个区块里含有多条交易trasaction（缩写为tx）构成的链表。图2下方有一个WorldState世界状态，这其实是为了提升性能用的。比如，key1共交易了10000次，为了获取它的当前状态值，需要正向执行这10000次交易，这就得不偿失了。如果这1万次交易里，每次新交易执行完，都同步更新一个数据库（在fabric里用的是levelDB），这样查询当前状态时，只需要查询该数据库即可，如图3所示。 ![](/2018/05/fabric-levelDB状态数据库-2.jpg) 图3-fabric levelDB状态数据库 图3中，区块链帐本是在FileSystem文件系统中保存的，而Level DB存放世界状态。

## 1.2 智能合约smart contract

区块链的发展过程中，一般1.0时代就是数字货币时代，代表是比特币，而2.0时代就是智能合约（现在是3.0时代，各种联盟链即为代表）。 智能合约是运行在区块链上的模块化、可重用的自动执行脚本，有了它我们就可以完成复杂的业务逻辑，例如同一个区块链上有多份合约，而每份合约可以约定不同的参与者（企业或者相关方）。也可以指定每份合约里每个子命令做一批特定的事，大家可以把它想象成关系数据库里的事务。如图4所示，我们可以在合约里指定允许哪些企业的节点可以参与到交易流程中来（在fabric里这叫共识策略）。 ![](/2018/05/unnamed-file-1-4.png) 图4-智能合约图示 在fabric中，智能合约叫做chaincode，它有6个状态，如下所示：

*   Install → Instantiate → invocable → Upgrade → Deinstantiate → Uninstall.

实际上智能合约就是一段代码，fabric官方认可的是GO语言。首先我们需要把合约代码上传到区块链上，这一步的状态就叫Install。 接着，需要做初始化操作。比如，现在的数据是存放在mysql中的，那么上线时需要用Instantiate把数据迁移至链上，这也算初始化。初始化后，chaincode就进入invocable可调用状态了。 通用我们可以通过CLI命令行或者程序里用SDK调用合约（v1.1前还有RestApi调用，现已放弃）。 联盟链由于跨多家企业、多个地区甚至国家，很难使得合约保持一致的版本，因此，每个合约都有版本号。而版本升级时，就是Upgrade状态。 最后两个状态对应着合约下链。 智能合约可以在供应链等较复杂的业务场景下起到很大的作用，如下面的图5所示： ![](/2018/05/unnamed-file-1-2.jpg) 图5-智能合约技术的应用示意

## 1.3 数据一致性（共识算法）

既然区块链是一个去中心化的分布式系统，那么自然只能通过投票来决定一致性了：少数服从多数。当然，多少算多数呢？不同的共识算法下，结果并不相同。比如paxos算法（参见笔者的《[paxos算法如何容错的–讲述五虎将的实践](https://www.taohui.pub/paxos%e7%ae%97%e6%b3%95%e5%a6%82%e4%bd%95%e5%ae%b9%e9%94%99%e7%9a%84-%e8%ae%b2%e8%bf%b0%e4%ba%94%e8%99%8e%e5%b0%86%e7%9a%84%e5%ae%9e%e8%b7%b5/)》）就是超过一半，而PBFT则需要三分之二以上。 这里有一个拜占庭将军问题需要注意，如何理解该问题可以参见这份翻译过的[The\_Part-Time\_Parliament(Paxos算法中文翻译)](http://www.taohui.pub/wp-content/uploads/2018/05/The_Part-Time_ParliamentPaxos算法中文翻译-3.pdf)文档。简言之，就是投票的拜占庭将军（服务器）们有2种不可靠的形式。第一是迟钝（数据包延迟）、失忆（数据包丢失以及数据包重发）、失踪（服务器宕机）等不含背叛的行为，第二则是有将军是间谍（服务器被攻破）。如paxos这样的算法属于第一种，Fault-tolerance，它不能容忍服务器上有恶意代码；而如PBFT(Practical Byzantine Fault Tolerance)这样的算法是第二类，Byzantine-Fault-tolerance，它能够容忍一定数量的拜占庭将军节点存在，如PBFT、SBFT、RBFT算法等。 第二类Byzantine-Fault-tolerance共识算法虽然看上去很美，但并不成熟，特别是性能低下，比如PBFT是一个多项式复杂度的算法O(N^2)，节点过多时（大于100）性能急骤下降。第一类通常是O(N)复杂度，在某些场景下使用效果还不错，比如fabric v1.1的kafka共识机制就是这样的算法，下文我们会详述。 像比特币、以太坊等采用的共识算法又有所不同，例如比特币的POW工作量证明算法，它定义一小时内（通过调整运算难度实现，比如调整近似程度）有一个lucky node节点，该节点是通过证明自身的努力（hash值逆解）而幸运选出，选出后它就可以为这段时间的交易做决定（似乎挺像总统选举^\_^）。详情参见我这篇文章：[《区块链技术学习笔记》](https://www.taohui.pub/%e5%8c%ba%e5%9d%97%e9%93%be%e6%8a%80%e6%9c%af%e5%ad%a6%e4%b9%a0%e7%ac%94%e8%ae%b0/)

## 1.4 非对称加密

区块链通过非对称加密技术实现身份验证与数据加密。其实就是我们日常在用的SSL技术。 为了方便理解，我们需要先介绍PKI(Public Key Infrastructure)，它是一种遵循标准的利用公钥加密技术为电子商务的开展提供一套安全基础平台的技术和规范。有一个CA（Certificate Authority）权威机构负责向用户（包括服务提供者与使用者）提供数字证书，包括公钥与私钥，同时CA机构还需要提供一个CRL（Certificate Revocation List）证书吊销列表，如下面的图6所示。 ![](/2018/05/PKI证书示意-1.png) 图6-CA机构颁发数字证书以及提供CAL 这样，区块链可以通过PKI体系实现安全认证。PKI有三个关键点，我们下面详述。

### 1.4.1 数字证书 Digital Certificate

比如Mary Morris符合X.509规范的数字证书里，其Subject属性里就含有她的信息，包括国家C=US、所属的州或者省份ST=Michigan、所在城市L=Detroit、所属单位O=Mitchesll Cars、其他信息OU=Manufacturing、公用信息CN=Mary Morris/UID=123456等，也含有其他信息，如下面的图7所示。 ![](/2018/05/PKI数字证书-2.png) 图7-PKI数字证书

### 1.4.2 公钥与私钥

CA颁发了两个证书：公钥与私钥，其中，私钥仅服务提供者保存，而公钥则可被所有人（服务使用者）保存。 所谓非对称加密，就是公钥加密的消息仅私钥可以解密；同理，私钥加密的消息，仅公钥可以解密。对应于前者，可以实现客户端访问服务器时加密消息，例如访问安全级别高的页面时提交的表单信息都需要用公钥加密，确保只有服务器才能解密网络报文。对应于后者，则可实现签名功能，如下面的图8所示。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/PKI公私钥-1.png) 图8-PKI中私钥签名后用公钥验签名 图8中Mary Morris用私钥对一段信息的内容（若内容过大则可先HASH后获得小点的字符串）加密后，生成签名附加在消息中。接收者可从CA机构获取到公钥，用公钥解密签名后，再与内容比对，以确定消息是否来自MaryMorris及内容是否被篡改。对于文件来说也是一样，小文件直接加密，大文件先生成hash再对hash加密，如下面的图9所示。   ![](/2018/05/unnamed-file-2-2.jpg) 图9-对文件的签名

### 1.4.3 证书信任链

CA证书分为两类：RCA(Root CA)根证书以及ICA（Intermediate CA）中间证书。这些证书由RCA开始构成一个证书信任链，如下面的图10所示。 ![](/2018/05/PKI证书信任链-1.png) 图10-CA证书信任链条 有许多CA证书权威机构，各自有其RCA。如果RCA得不到信任，那么其下的ICA也无法认证通过。 当然，自己的服务器也可以生成RCA。 在Fabric里，允许不同的企业使用不同的RCA，也可以使用相同的RCA和不同的ICA。这与下文中的MSP密切相关。

## 1.5 小结

我们来总结下区块链，它主要是为了解决社会上的信任问题而存在的，为此，它付出了沉重的性能、可用性代价。它怎么做到的呢？通过4点实现：1、数据到处存放；2、操作记录不可更改；3、传输数据可信；4、业务脚本约束。 那么，这个信任问题的解决，带来了2个非功能性的约束：数据一致性和可用性。其中可用性包括两点：1、交易在可接受的时间内达成。比如比特币的分叉就会造成严重问题。2、吞吐量达标。而比特币每秒只能有7次交易，这显然太低了。

# 2、fabric核心概念

hyperledger fabric符合上面说过的区块链的所有特性。我们必须先了解它的一些概念，才能进一步理解其架构设计。由于英文资料居多，所以这些概念我都以英文描述为准：

*   chaincode：智能合约，上文已提到。每个chaincode可提供多个不同的调用命令。
*   transaction：交易，每条指令都是一次交易。
*   world state：对同一个key的多次交易形成的最终value，就是世界状态。
*   endorse：背书。金融上的意义为：指持票人为将票据权利转让给他人或者将一定的票据权利授予他人行使，而在票据背面或者粘单上记载有关事项并签章的行为。通常我们引申为对某个事情负责。在我们的共识机制的投票环节里，背书意味着参与投票。
*   endorsement policy：背书策略。由智能合约chaincode选择哪些peer节点参与到背书环节来。
*   peer：存放区块链数据的结点，同时还有endorse和commit功能。
*   channel：私有的子网络，事实上是为了隔离不同的应用，一个channel可含有一批chaincode。
*   PKI：Public Key Infrastructure，一种遵循标准的利用公钥加密技术为电子商务的开展提供一套安全基础平台的技术和规范。
*   MSP：Membership Service Provider，联盟链成员的证书管理，它定义了哪些RCA以及ICA在链里是可信任的，包括定义了channel上的合作者。
*   org：orginazation，管理一系列合作企业的组织。

## 2.1 开发概念

fabric联盟链的开发人员主要分为三类：底层是系统运维，负责系统的部署与维护；其次是组织管理人员，负责证书、MSP权限管理、共识机制等；最后是业务开发人员，他们负责编写chaincode、创建维护channel、执行transaction交易等，如下面的图11所示。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/fabric分层视角-1.png) 图11-fabric技术人员的分层 fabric大致分为底层的网络层、权限管理模块、区块链应用模块，通过SDK和CLI对应用开发者提供服务，如下面的图12所示。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/fabric整体架构-1.png) 图12-fabric开发模块图 我们的开发流程主要包括写智能合约，以及通过SDK调用智能合约，及订阅各类事件，如图13所示。 ![](/2018/05/fabric开发流程-2.jpg) 图13-开发环节

## 2.2 MSP

每个管理协作企业的ORG组织都可以拥有自己的MSP。如下图14所示，组织ORG1拥有的MSP叫ORG1.MSP，而组织ORG2业务复杂，所以维护了3个MSP。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/MSP与组织间关系-1.png) 图14-ORG可管理自己的MSP MSP出现在两个地方：在channel上有一个全局的MSP，而每个peer、orderer、client等角色上都维护有本地的局部MSP，如图15所示。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/MSP本地与channel-3.png) 图15-在channel上的Global MSP以及在参与角色上的Local MSP 本地MSP只保存有Global MSP上的子集，内容保存在本地文件系统上，而全局MSP可在逻辑上认为是配置在系统上的，它实际也在每个参与者上保存一份拷贝，但会维持一致性。 MSP也分级，如图16中所示，底层的network MSP负责网络层的准入，其MSP由ORG1拥有，而上面的某个channel的MSP则由ORG1和ORG2共同管理。 ![](http://www.taohui.pub/wp-content/uploads/2018/05/MSP-levels-1.png) 图16-MSP是分级的 一个MSP下含有以下结构，如图17所示。 ![](/2018/05/MSP-structure-1.png) 图17-MSP结构 可见，MSP结构包括：

*   RCA根证书
*   ICA中间证书
*   OU组织单位
*   管理员证书
*   RCL吊销证书列表
*   结点上的具体证书
*   存储私钥的keystore
*   TLS的根证书与中间证书

# 3、fabric交易提交流程

 

## 3.1 peer结点的部署

peer结点上保存有账本ledger以及智能合约，如下图所示： ![](http://www.taohui.pub/wp-content/uploads/2018/05/fabric-blockchain-network-2.png) channel是一个逻辑概念，可以通过MSP隔离全网不同组织的参与者，如下图所示： ![](http://www.taohui.pub/wp-content/uploads/2018/05/fabric-peer与channel-3.png) 当有多方参与者时，例如4个org组织、8个peer结点时，其中channel连接了P1、P3、P5、P7、P8这五个节点，其他3个节点加入了其他channel，其部署图如下所示： ![](http://www.taohui.pub/wp-content/uploads/2018/05/fabric-peer与不同组织-3.png) 加入MSP来管理身份时，如P1和P2由ORG1.MSP管理，而P3和P4的证书则由ORG2.MSP管理，他们共同使用一个channel，则如下图所示： ![](/2018/05/fabric-peer与身份-2.png)

## 3.2 交易的执行流程

去中心化的设计，必然需要通过投票（多数大于少数）来维持数据一致性，而任何投票都必须经历以下三个过程：

1.  有一方先提出议案proposal，该议案有对应的一批投票者需要对该结果背书，这些投票者依据各自的习惯投票，并将结果反馈；
2.  统计投票结果，若获得多数同意，才能进行下一步；
3.  将获得多数同意的议案记录下来，且公之于众。

而这三步fabric当然也少不了，当然它的称法就有所不同，其对应的三步如下：

1.  由client上的CLI或者SDK进行proposal议案的提出。client会依据智能合约chaincode根据背书策略endorse policy决定把proposal发往哪些背书的peer节点，而peer节点进行投票，client汇总各背书节点的结果；
2.  client将获得多数同意的议案连同各peer的背书（包括其投票结果以及背书签名）交给orderring service，而orderer会汇总各client递交过来的trasaction交易，排序、打包。
3.  orderer将交易打包成区块block，然后通知所有commit peer，各peer各自验证结果，最后将区块block记录到自己的ledger账本中。

我们看一个具体的例子，若channel上有三个peer背书者，client提交流程如下图所示： ![](/2018/05/Illustration-of-one-possible-transaction-flow-2.png) 详细解释下上图的流程：

1.  首先，client发起一个transaction交易，含有<clientID, chaincodeID, txPayLoad, timestamp, clientSig>等信息，指明了3W要素：消息是谁who在什么时间when发送了什么what。该消息根据chaincode中的背书策略，发向EP1、EP2、EP3这三个peer节点。
2.  这三个peer节点模拟执行智能合约，并将结果及其各自的CA证书签名发还client。client收集到足够数量的结果后再进行下一步。
3.  client将含背书结果的tx交易发向ordering service。
4.  ordering service将打包好的block交给committing peer CP1以及EP1、EP2、EP3这三个背书者，背书者此时会校验结果并写入世界状态以及账本中。同时，client由于订阅了消息，也会收到通知。 如果我们从编程的角度来看，则流程会更清楚：

![](/2018/05/fabric请求连接图-1.png) 参见上图，A是我们的应用程序，其步骤如下：

1.  A首先连接到peer。
2.  A调用chaincode发起proposal；与此同时，P1收到后先模拟执行，再产生结果返回给A。
3.  A收到各peer返回的结果。
4.  A向O1发起交易；与此同时，O1产生区块后会通知peer，而peer会更新其账本。
5.  最后通过订阅事件A收到了结果。

最后再细看下这三个阶段。

### 3.2.1 proposal提案阶段

![](/2018/05/fabric共识提案阶段-1.png) 可以看到，A1发出的<T1, P>，收到了<T1, R1, E1>和<T1, R2, E2>两个结果。

### 3.2.2 package打包阶段

![](/2018/05/fabric共识打包阶段-1.png) O1在一个channel上会收到许多T交易，它会将T排序，在达到block的最大大小（一般应配1M以下，否则性能下降严重，kafka擅长处理小点的消息）或者达到超时时间后，打成区块P2。

### 3.2.3 验证阶段

![](/2018/05/fabric共识验证阶段-2.png) O1将含有多条交易T打成区块的B2发往各peer节点，而P1和P2将B2加入各自的L账本中。

# 4、小结

本文偏重于概念的解释，由于篇幅所限，未涉及fabric的系统搭建（请参考笔者的这篇文章《[区块链开源实现fabric快速部署及CLI体验](https://www.taohui.pub/%e5%8c%ba%e5%9d%97%e9%93%be%e5%bc%80%e6%ba%90%e5%ae%9e%e7%8e%b0fabric%e5%bf%ab%e9%80%9f%e9%83%a8%e7%bd%b2%e5%8f%8acli%e4%bd%93%e9%aa%8c/)》），也未描述共识算法在异常情况下如何维持一致性，这留待下一篇文章解决。fabric的许多思想是值得我们进一步研究的，其优秀的实现可以帮助我们通过fabric获得区块链在信任创新上的思路。