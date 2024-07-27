# Fisco Bcos 2.9.1

FISCO BCOS 平台是金融区块链合作联盟（简称：金链盟）开源工作组以金融业务实践为参考样本，在 BCOS 开源平台基础上进行模块升级与功能重塑，深度定制的安全可控、适用于金融行业且完全开源的区块链底层平台

## 1. 安装

### 1.1 WeBASE

WeBASE（WeBank Blockchain Application Software Extension） 是在区块链应用和FISCO-BCOS节点之间搭建的一套通用组件。围绕交易、合约、密钥管理，数据，可视化管理来设计各个模块，开发者可以根据业务所需，选择子系统进行部署。WeBASE屏蔽了区块链底层的复杂度，降低开发者的门槛，大幅提高区块链应用的开发效率，包含节点前置、节点管理、交易链路，数据导出，Web管理平台等子系统

#### 1.1.1 下载解压

```bash
cd /opt/software
wget https://osp-1257653870.cos.ap-guangzhou.myqcloud.com/WeBASE/releases/download/v1.5.5/webase-deploy.zip
unzip webase-deploy.zip
mv webase-deploy /opt/
```

#### 1.1.2 修改配置

```bash
cd /opt/webase-deploy
vi common.properties
```

```conf
[common]
# WeBASE子系统的最新版本（v3.1.0版本）
webase.web.version=v1.5.5
webase.mgr.version=v1.5.5
webase.sign.version=v1.5.5
webase.front.version=v1.5.5

#####################################################################
# if use [installDockerAll] to install WeBASE by docker
# if use [installAll] or [installWeBASE], ignore configuration here

# 1: enable mysql in docker
# 0: mysql run in host, required fill in the configuration of webase-node-mgr and webase-sign
docker.mysql=1

# if [docker.mysql=1], mysql run in host (only works in [installDockerAll])
# run mysql 5.6 by docker
docker.mysql.port=23306
docker.mysql.password=123456
#####################################################################

# 节点管理子系统mysql数据库配置
mysql.ip=localhost
mysql.port=3306
mysql.user=dbUsername
mysql.password=dbPassword
mysql.database=webasenodemanager

# 签名服务子系统mysql数据库配置
sign.mysql.ip=localhost
sign.mysql.port=3306
sign.mysql.user=dbUsername
sign.mysql.password=dbPassword
sign.mysql.database=webasesign

# 节点前置子系统h2数据库名
front.h2.name=webasefront
front.org=fisco

# WeBASE管理平台服务端口
web.port=5000
web.h5.enable=1

# 节点管理子系统服务端口
mgr.port=5001

# 节点前置子系统端口
front.port=5002

# 签名服务子系统端口
sign.port=5004

# 是否使用已有的链（yes/no）
if.exist.fisco=no

# 搭建新链时需配置[if.exist.fisco=no]
# 节点监听Ip
node.listenIp=127.0.0.1
# 节点p2p端口
node.p2pPort=30300
# 节点channel端口
node.channelPort=20200
# 节点rpc端口
node.rpcPort=8545

# SDK连接加密类型 （0: ECDSA SSL, 1: 国密SSL,支持ssl）
encrypt.type=0
encrypt.sslType=0

# FISCO-BCOS版本（v.2.9.1）
fisco.version=2.9.1
# 搭建节点个数（默认两个）
node.counts=nodeCounts

# 使用已有链时需配置[if.exist.fisco=yes]
# 已有链节点rpc端口列表
fisco.dir=/data/app/nodes/127.0.0.1

# 选择fisco.dir下面任意node中的就行（其中包含ca.crt, node.crt and node. Key）
node.dir=node0

```

#### 1.1.3 部署并启动所有服务

```bash
python3 deploy.py installAll
```

`不要用sudo执行脚本，sudo会导致无法获取当前用户的环境变量如JAVA_HOME。`

```bash
cd /opt/webase-deploy
# 一键部署
python3 deploy.py installAll    # 部署并启动所有服务
python3 deploy.py stopAll       # 停止一键部署的所有服务
python3 deploy.py startAll      # 启动一键部署的所有服务
# 各子服务启停
python3 deploy.py startNode     # 启动FISCO-BCOS节点
python3 deploy.py stopNode      # 停止FISCO-BCOS节点
python3 deploy.py startWeb      # 启动WeBASE-Web
python3 deploy.py stopWeb       # 停止WeBASE-Web
python3 deploy.py startManager  # 启动WeBASE-Node-Manager
python3 deploy.py stopManager   # 停止WeBASE-Node-Manager
python3 deploy.py startSign     # 启动WeBASE-Sign
python3 deploy.py stopSign      # 停止WeBASE-Sign
python3 deploy.py startFront    # 启动WeBASE-Front
python3 deploy.py stopFront     # 停止WeBASE-Front
```

> 访问地址：http://localhost:5000 ，默认账号为 admin ，默认密码为 Abcd1234 ，密码不能包含特殊字符。

#### 1.1.4 设置节点存储方式

通过群组配置文件group.[group_id].ini的storage配置项可配置MySQL为存储介质，如本地节点0的群组1的配置文件位置是`nodes/127.0.0.1/node0/group.1.ini`

```bash
cd /opt/webase-deploy/nodes/127.0.0.1/node0/conf
```

```bash
# 修改存储类型为mysql
sed -i 's/type=rocksdb/type=mysql/g' ~/fisco/nodes/127.0.0.1/node*/conf/group.1.ini

# 配置数据库用户名和密码
sed -i 's/db_username=/db_username=root/g' ~/fisco/nodes/127.0.0.1/node*/conf/group.1.ini
sed -i 's/db_passwd=/db_passwd=123456/g' ~/fisco/nodes/127.0.0.1/node*/conf/group.1.ini
sed -i 's/db_name=/db_name=db/g' ~/fisco/nodes/127.0.0.1/node*/conf/group.1.ini
```

重启节点

```bash
cd /opt/webase-deploy
python3 deploy.py stopNode
python3 deploy.py startNode
```


### 1.2 命令行交互控制台

命令行交互控制台 (简称“控制台”) 是FISCO BCOS 2.0重要的交互式客户端工具，它通过 Java SDK 与区块链节点建立连接，实现对区块链节点数据的读写访问请求。控制台拥有丰富的命令，包括查询区块链状态、管理区块链节点、部署并调用合约等。此外，控制台提供一个合约编译工具，用户可以方便快捷的将Solidity合约文件编译为Java合约文件

控制台`2.6+`基于Java SDK实现, 控制台`1.x`系列基于Web3SDK实现。

#### 1.2.1 下载解压

```bash
cd /opt/software
curl -LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh
tar -zxvf console.tar.gz -C /opt/fisco
```

> 注意：默认下载的控制台内置0.4.25版本的solidity编译器，用户需要编译0.5或者0.6版本的合约时，可以通过下列命令获取内置对应编译器版本的控制台

```bash
# 0.5	
curl -#LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh -v 0.5	
# 0.6	
curl -#LO https://github.com/FISCO-BCOS/console/releases/download/v2.9.2/download_console.sh && bash download_console.sh -v 0.6	
```


#### 1.2.2 配置控制台

1. 区块链节点和证书的配置

将conf目录下的config-example.toml文件重命名为config.toml文件。配置config.toml文件，其中添加注释的内容根据区块链节点配置做相应修改，如果搭链时设置的channel_listen_ip(若节点版本小于v2.3.0，查看配置项listen_ip)为127.0.0.1或者0.0.0.0，channel_port为`20200`， 则config.toml配置不用修改

```bash
cd /opt/fisco/console
cp -n conf/config-example.toml conf/config.toml
```

将节点sdk目录下的ca.crt、sdk.crt和sdk.key文件拷贝到conf目录下

```bash
cd /opt/fisco/console
cp -r /opt/webase-deploy/nodes/127.0.0.1/sdk/* conf/
```

FISCO-BCOS 2.5及之后的版本，添加了SDK只能连本机构节点的限制，操作时需确认拷贝证书的路径，否则建联报错

#### 1.2.3 启动服务

```bash
cd /opt/fisco/console && bash start.sh
```

#### 1.2.4 控制台命令列表

```bash
addObserver                               Add an observer node
addPeers                                  Add more connected peers to the node p2p network
addSealer                                 Add a sealer node
call                                      Call a contract by a function and parameters
callByCNS                                 Call a contract by a function and parameters by CNS
chargeGas                                 Charge specified gas for the given account
create                                    Create table by sql
deductGas                                 Deduct specified gas from the given account
delete                                    Remove records by sql
deploy                                    Deploy a contract on blockchain
deployByCNS                               Deploy a contract on blockchain by CNS
desc                                      Description table information
erasePeers                                Remove connected peers of the node p2p network
quit([quit, q, exit])                     Quit console
freezeAccount                             Freeze the account
freezeContract                            Freeze the contract
generateGroup                             Generate a group for the specified node
generateGroupFromFile                     Generate group according to the specified file
getAccountStatus                          GetAccountStatus of the account
getAvailableConnections                   Get the connection information of the nodes connected with the sdk
getBatchReceiptsByBlockHashAndRange       Get batched transaction receipts according to block hash and the transaction range
getBatchReceiptsByBlockNumberAndRange     Get batched transaction receipts according to block number and the transaction range
getBlockByHash                            Query information about a block by hash
getBlockByNumber                          Query information about a block by number
getBlockHashByNumber                      Query block hash by block number
getBlockHeaderByHash                      Query information about a block header by hash
getBlockHeaderByNumber                    Query information about a block header by block number
getBlockNumber                            Query the number of most recent block
getCode                                   Query code at a given address
getConsensusStatus                        Query consensus status
getContractStatus                         Get the status of the contract
getCryptoType                             Get the current crypto type
getCurrentAccount                         Get the current account info
getDeployLog                              Query the log of deployed contracts
getGroupConnections                       Get the node information of the group connected to the SDK
getGroupList                              Query group list
getGroupPeers                             Query nodeId list for sealer and observer nodes
getNodeIDList                             Query nodeId list for all connected nodes
getNodeInfo                               Query the specified node information.
getNodeVersion                            Query the current node version
getObserverList                           Query nodeId list for observer nodes.
getPbftView                               Query the pbft view of node
getPeers                                  Query peers currently connected to the client
getPendingTransactions                    Query pending transactions
getPendingTxSize                          Query pending transactions size
getSealerList                             Query nodeId list for sealer nodes
getSyncStatus                             Query sync status
getSystemConfigByKey                      Query a system config value by key
getTotalTransactionCount                  Query total transaction count
getTransactionByBlockHashAndIndex         Query information about a transaction by block hash and transaction index position
getTransactionByBlockNumberAndIndex       Query information about a transaction by block number and transaction index position
getTransactionByHash                      Query information about a transaction requested by transaction hash
getTransactionByHashWithProof             Query the transaction and transaction proof by transaction hash
getTransactionReceipt                     Query the receipt of a transaction by transaction hash
getTransactionReceiptByHashWithProof      Query the receipt and transaction receipt proof by transaction hash
grantCNSManager                           Grant permission for CNS by address
grantCharger                              Grant the charge permission for the given account
grantCommitteeMember                      Grant the account committee member
grantContractStatusManager                Grant contract authorization to the user
grantContractWritePermission              Grant the account the contract write permission.
grantDeployAndCreateManager               Grant permission for deploy contract and create user table by address
grantNodeManager                          Grant permission for node configuration by address
grantOperator                             Grant the account operator
grantSysConfigManager                     Grant permission for system configuration by address
grantUserTableManager                     Grant permission for user table by table name and address
insert                                    Insert records by sql
listAbi                                   List functions and events info of the contract.
listAccount                               List the current saved account list
listCNSManager                            Query permission information for CNS
listChargers                              List the accounts that have the permission to charge/deduct gas
listCommitteeMembers                      List all committee members
listContractStatusManager                 List the authorization of the contract
listContractWritePermission               Query the account list which have write permission of the contract.
listDeployAndCreateManager                Query permission information for deploy contract and create user table
listDeployContractAddress                 List the contractAddress for the specified contract
listNodeManager                           Query permission information for node configuration
listOperators                             List all operators
listSysConfigManager                      Query permission information for system configuration
listUserTableManager                      Query permission for user table information
loadAccount                               Load account for the transaction signature
newAccount                                Create account
queryCNS                                  Query CNS information by contract name and contract version
queryCommitteeMemberWeight                Query the committee member weight
queryGroupStatus                          Query the status of the specified group of the specified node
queryPeers                                Query the configured connected node list of the p2p network
queryRemainGas                            Query remain gas for the given account
queryThreshold                            Query the threshold
queryVotesOfMember                        Query votes of a committee member.
queryVotesOfThreshold                     Query votes of updateThreshold operation
recoverGroup                              Recover the specified group of the specified node
registerCNS                               RegisterCNS information for the given contract
removeGroup                               Remove the specified group of the specified node
removeNode                                Remove a node
revokeCNSManager                          Revoke permission for CNS by address
revokeCharger                             Revoke the charge permission from the given account
revokeCommitteeMember                     Revoke the account from committee member
revokeContractStatusManager               Revoke contract authorization to the user
revokeContractWritePermission             Revoke the account the contract write permission
revokeDeployAndCreateManager              Revoke permission for deploy contract and create user table by address
revokeNodeManager                         Revoke permission for node configuration by address
revokeOperator                            Revoke the operator
revokeSysConfigManager                    Revoke permission for system configuration by address
revokeUserTableManager                    Revoke permission for user table by table name and address
switch([s])                               Switch to a specific group by group ID
select                                    Select records by sql
setSystemConfigByKey                      Set a system config value by key
startGroup                                Start the specified group of the specified node
stopGroup                                 Stop the specified group of the specified node
unfreezeAccount                           Unfreeze the account
unfreezeContract                          Unfreeze the contract
update                                    Update records by sql
updateCommitteeMemberWeight               Update the committee member weight
updateThreshold                           Update the threshold
```

#### 1.2.5 合约编译工具

```bash
# 若控制台版本大于等于2.8.0
bash sol2java.sh -p net.xzh.fisco
```

运行成功之后，将会在console/contracts/sdk目录生成java、abi和bin目录如下

```bash
├── abi # 编译生成的abi目录，存放solidity合约编译的abi文件
│   ├── Crypto.abi
│   ├── HelloWorld.abi
│   ├── KVTableTest.abi
│   ├── ShaTest.abi
│   ├── Table.abi
│   └── TableTest.abi
├── bin # 编译生成的bin目录，存放solidity合约编译的bin文件
│   ├── Crypto.bin
│   ├── HelloWorld.bin
│   ├── KVTableTest.bin
│   ├── ShaTest.bin
│   ├── Table.bin
│   └── TableTest.bin
└── java # 存放编译的包路径及Java合约文件
    └── net
        └── xzh
            └── fisco
                ├── Crypto.java
                ├── HelloWorld.java
                ├── KVTableTest.java
                ├── ShaTest.java
                ├── Table.java
                └── TableTest.java
```