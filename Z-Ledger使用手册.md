





# 一.前提准备

- 中标麒麟 V7.0
- 已安装 Docker 及 Docker Compose
- 已下载到 Docker 相关镜像

如果以上环境未准备完全，请参照《Z-Ledger安装手册》进行安装。

# 二.启动 Z-Ledger 网络

## 2.1 配置启动文件

Z-Ledger 网络在启动之前，需要提前生成一些用于启动的配置文件，主要包括 MSP 相关文件（msp/*）、TLS 相关文件（tls/*）、系统通道初始区块（orderer.genesis.block）、新建应用通道交易文件（businesschannel.tx）、锚节点配置更新交易文件（ Org1MSPanchors.tx 和 Org2MSPanchors.tx）等。各个文件的功能如下表：

| 配置                                                         | 存放节点              | 依赖配置           | 主要功能                                                 |
| ------------------------------------------------------------ | --------------------- | ------------------ | -------------------------------------------------------- |
| MSP 相关文件 msp/*                                           | Peer、Orderer、客户端 | crypto-comfig.yaml | 包括证书文件、签名私钥等，用于管理实体在网络中的身份信息 |
| TLS 相关文件 tls/*                                           | Peer、Orderer、客户端 | crypto-comfig.yaml | 如果网络中启用了 TLS，则节点需要准备 TLS 证书            |
| 系统通道初始区块文件（orderer.genesis.block）                | Orderer               | comfigtx.yaml      | 用于启动 Ordering 服务，配置网络中策略                   |
| 新建应用通道交易文件（businesschannel.tx）                   | 客户端                | comfigtx.yaml      | 用于新建应用通道，指定通道成员、访问策略等               |
| 锚节点配置更新交易文件<br />Org1MSPanchors.tx 和Org2MSPanchors.tx | 客户端                | comfigtx.yaml      | 用于配置通道中各组织的锚节点信息                         |

Z-Ledger 的 zhigui/z-ledger-tools 镜像包含了使用上述配置文件的 cyrptogen 和 configtxgen 工具，所以在使用上述配置文件和工具之前，我们得先启动 zhigui/z-ledger-tools 镜像，之后在该镜像启动的容器中来使用 cyrptogen 和 configtxgen 工具。

用以下命令来启动 zhigui/z-ledger-tools 镜像：

```bash
$ docker run -dit zhigui/z-ledger-tools:2.0
```

启动成功后通过以下命令进入容器:

```bash
$ docker exec -it [CONTAIN_ID] /bin/bash
```

进入容器后输入以下命令检查是否可正常使用 ：

```bash
$ cryptogen
usage: cryptogen [<flags>] <command> [<args> ...]

Utility for generating Hyperledger Fabric key material

Flags:
  --help  Show context-sensitive help (also try --help-long and --help-man).

Commands:
  help [<command>...]
    Show help.

  generate [<flags>]
    Generate key material

  showtemplate
    Show the default configuration template

  version
    Show version information

  extend [<flags>]
    Extend existing network
```

下文中使用 cyrptogen 和 configtxgen 工具均在容器内进行。

### 2.1.1 生成组织关系和身份证书

Z-Ledger 网络提供的是联盟链服务，联盟由多个组织构成，组织中的成员提供了节点服务来维护网络，并且通过身份来进行权限管理。因此，首先需要对各个组织和成员的关系进行规划，分别生成对应的身份证书文件，并部署到其对应的节点上。

Z-Ledger 项目提供了 cryptogen 工具（基于 crypto 标准库）实现自动化生成各个实体证书和私钥。这一过程首先依赖 crypto-config.yaml 配置文件。

crypto-config.yaml 配置文件的结构十分简单，支持定义两种类型（OrdererOrgs 和PeerOrgs）的若干组织。每个组织中又可以定义多个节点（Spec）和用户（User）。

一个示例的 crypto-config.yaml 配置文件内容如下，其中定义了一个 OrdererOrgs 类型的组织 Orderer，包括一个节点 orderer.example.com；两个 PeerOrgs 类型的组织 Org1 和Org2，分别包括 2 个节点的 1 个普通用户：

```yaml

OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    EnableNodeOUs: true
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    EnableNodeOUs: true
    Template:
      Count: 2
    Users:
      Count: 1

```

使用该配置文件，通过如下命令可以为 Z-Ledger 网络生成指定拓扑结构的组织和身份文件，存放到 crypto-config 目录下： 

```bash
$ cryptogen generate --config=./crypto-config.yaml --output ./crypto-config --crypto="SW/ECDSA/SHA2-256"
```



### 2.1.2 生成 Ordering 服务启动初始区块

Orderer 节点在启动时，可以指定使用提前生成的初始区块文件作为系统通道的初始配置。初始区块中包括了 Ordering 服务的相关配置信息，以及联盟信息。初始区块可以使用 configtxgen 工具进行生成。生成过程需要依赖 configtx.yaml 文件。

编写 configtx.yaml 配置文件可以参考 Z-Ledger 代码中（如 examples/e2e_cli 路径下或 sampleconfig 路径下）的示例。这里采用如下内容进行生成：

```yaml

Organizations:
    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        MSPDir: crypto-config/ordererOrganizations/example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('OrdererMSP.admin')"

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org1MSP.admin')"
        AnchorPeers:
            - Host: peer0.org1.example.com
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.example.com/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.peer', 'Org2MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org2MSP.admin')"
        AnchorPeers:
            - Host: peer0.org2.example.com
              Port: 9051

Capabilities:
    Channel: &ChannelCapabilities
        V1_4_3: true
        V1_3: false
        V1_1: false
    Orderer: &OrdererCapabilities
        V1_4_2: true
        V1_1: false
    Application: &ApplicationCapabilities
        V1_4_2: true
        V1_3: false
        V1_2: false
        V1_1: false
        
Application: &ApplicationDefaults
    Organizations:
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    Capabilities:
        <<: *ApplicationCapabilities
        
Orderer: &OrdererDefaults
    OrdererType: solo
    Addresses:
        - orderer.example.com:7050
    BatchTimeout: 2s
    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB

    Kafka:
        Brokers:
            - 127.0.0.1:9092
    EtcdRaft:
        Consenters:
            - Host: orderer.example.com
              Port: 7050
              ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
              ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
            - Host: orderer2.example.com
              Port: 7050
              ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
              ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
            - Host: orderer3.example.com
              Port: 7050
              ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
              ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
            - Host: orderer4.example.com
              Port: 7050
              ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
              ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
            - Host: orderer5.example.com
              Port: 7050
              ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
              ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"

Channel: &ChannelDefaults
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
    Capabilities:
        <<: *ChannelCapabilities

Profiles:

    TwoOrgsOrdererGenesis:
        <<: *ChannelDefaults
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
    TwoOrgsChannel:
        Consortium: SampleConsortium
        <<: *ChannelDefaults
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2
            Capabilities:
                <<: *ApplicationCapabilities

    SampleDevModeKafka:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: kafka
            Kafka:
                Brokers:
                    - kafka.example.com:9092

            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2

    SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            EtcdRaft:
                Consenters:
                    - Host: orderer.example.com
                      Port: 7050
                      ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                      ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/server.crt
                    - Host: orderer2.example.com
                      Port: 7050
                      ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                      ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer2.example.com/tls/server.crt
                    - Host: orderer3.example.com
                      Port: 7050
                      ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                      ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer3.example.com/tls/server.crt
                    - Host: orderer4.example.com
                      Port: 7050
                      ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                      ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer4.example.com/tls/server.crt
                    - Host: orderer5.example.com
                      Port: 7050
                      ClientTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
                      ServerTLSCert: crypto-config/ordererOrganizations/example.com/orderers/orderer5.example.com/tls/server.crt
            Addresses:
                - orderer.example.com:7050
                - orderer2.example.com:7050
                - orderer3.example.com:7050
                - orderer4.example.com:7050
                - orderer5.example.com:7050

            Organizations:
                - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2

```

该配置文件定义了两个模板：TwoOrgsOrdererGenesis 和 TwoOrgsChannel，其中前者可以用来生成 Ordering 服务的初始区块文件。

通过如下命令指定使用 configtx.yaml 文件中定义的 TwoOrgsOrdererGenesis 模板，来生成 Ordering 服务系统通道的初始区块文件。

```bash
# 设置环境变量为当前路径
$ FABRIC_CFG_PATH=./
$ configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./orderer.genesis.block
```

### 2.1.3 生成新建应用通道的配置交易

新建应用通道时，需要事先准备好配置交易文件，其中包括了属于该通道的组织结构信息。这些信息会写入到该应用通道的初始区块中。

同样需要提前编写好 configtx.yaml 配置文件，之后可以使用 configtxgen 工具来生成新建通道的配置交易文件。为了后续命令使用方便，将新建应用通道名称 businesschannel 复制到环境变量CHANNEL_NAME 中：

```bash
$ CHANNEL_NAME=businesschannel
```

之后采用如下命令指定使用 configtx.yaml 配置文件中的 TwoOrgsChannel 模板，来生成新建通道的配置交易文件。TwoOrgsChannel 模板指定了 Org1 和 Org2 都属于后面新建的应用通道：

```bash
$ configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./businesschannel.tx -channelID ${CHANNEL_NAME}
```

### 2.1.4 生成 docker-compose 文件

节点在启动时会依赖一些配置，为方便管理，我们使用 docker-compose 来启动 Docker 镜像。所以我们接下来需要编写三个docker-compose文件：docker-compose-cli.yaml、docker-compose-base.yaml 、docker-compose-ca.yaml 和 peer-base.yaml。

其中 docker-compose-cli.yaml 用来启动 zhigui/z-ledger-tools 镜像的，我们后续会用这个镜像启动的容器充当客户端。docker-compose-base.yaml 和 peer-base.yaml 用来启动 orderer 和 peer 节点。

docker-compose-cli.yaml文件内容如下:

```yaml
version: '2'

volumes:
  orderer.example.com:
  peer0.org1.example.com:
  peer1.org1.example.com:
  peer0.org2.example.com:
  peer1.org2.example.com:

networks:
  byfn:

services:

  orderer.example.com:
    extends:
      file:  ./docker-compose-base.yaml
      service: orderer.example.com
    container_name: orderer.example.com
    networks:
      - byfn

  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file:  ./docker-compose-base.yaml
      service: peer0.org1.example.com
    networks:
      - byfn

  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    extends:
      file:  ./docker-compose-base.yaml
      service: peer1.org1.example.com
    networks:
      - byfn

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    extends:
      file:  ./docker-compose-base.yaml
      service: peer0.org2.example.com
    networks:
      - byfn

  peer1.org2.example.com:
    container_name: peer1.org2.example.com
    extends:
      file:  ./docker-compose-base.yaml
      service: peer1.org2.example.com
    networks:
      - byfn

  cli:
    container_name: cli
    image: zhigui/z-ledger-tools:2.0
    tty: true
    stdin_open: true
    environment:
      - SYS_CHANNEL=byfn-sys-channel
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
#      - FABRIC_LOGGING_SPEC=DEBUG
      - FABRIC_LOGGING_SPEC=INFO
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_TLS_ENABLED=false
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: /bin/bash
    volumes:
        - /var/run/:/host/var/run/
        - ./chaincode_example02/:/opt/gopath/src/github.com/chaincode_example02
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./scripts:/opt/gopath/src/github.com/hyperledger/fabric/peer/scripts/
        - ./businesschannel.tx:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts/businesschannel.tx
    depends_on:
      - orderer.example.com
      - peer0.org1.example.com
      - peer1.org1.example.com
      - peer0.org2.example.com
      - peer1.org2.example.com
    networks:
      - byfn

```

peer-base.yaml 内容如下：

```yaml
version: '2'

services:
  peer-base:
    image: zhigui/z-ledger-peer:2.0
    environment:
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=config_byfn
      - FABRIC_LOGGING_SPEC=INFO
#      - FABRIC_LOGGING_SPEC=DEBUG
      - CORE_PEER_TLS_ENABLED=false
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start

  orderer-base:
    image: zhigui/z-ledger-orderer:2.0
    environment:
      - FABRIC_LOGGING_SPEC=INFO
#      - FABRIC_LOGGING_SPEC=DEBUG
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=false
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
      - ORDERER_KAFKA_TOPIC_REPLICATIONFACTOR=1
      - ORDERER_KAFKA_VERBOSE=true
      - ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer


```

docker-compose-base.yaml  文件内容如下：

```yaml
version: '2'

services:

  orderer.example.com:
    container_name: orderer.example.com
    extends:
      file: peer-base.yaml
      service: orderer-base
    volumes:
        - ./orderer.genesis.block:/var/hyperledger/orderer/orderer.genesis.block
        - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp:/var/hyperledger/orderer/msp
        - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/tls/:/var/hyperledger/orderer/tls
        - orderer.example.com:/var/hyperledger/production/orderer
    ports:
      - 7050:7050

  peer0.org1.example.com:
    container_name: peer0.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org1.example.com
      - CORE_PEER_ADDRESS=peer0.org1.example.com:7051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:7051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org1.example.com:7052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:7052
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org1.example.com:8051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org1.example.com:/var/hyperledger/production
    ports:
      - 7051:7051

  peer1.org1.example.com:
    container_name: peer1.org1.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org1.example.com
      - CORE_PEER_ADDRESS=peer1.org1.example.com:8051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:8051
      - CORE_PEER_CHAINCODEADDRESS=peer1.org1.example.com:8052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:8052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.example.com:8051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org1.example.com:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org1.example.com:/var/hyperledgerproduction

    ports:
      - 8051:8051

  peer0.org2.example.com:
    container_name: peer0.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org2.example.com
      - CORE_PEER_ADDRESS=peer0.org2.example.com:9051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:9051
      - CORE_PEER_CHAINCODEADDRESS=peer0.org2.example.com:9052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:9052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.example.com:9051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer1.org2.example.com:10051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer0.org2.example.com:/var/hyperledgerproduction
    ports:
      - 9051:9051

  peer1.org2.example.com:
    container_name: peer1.org2.example.com
    extends:
      file: peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer1.org2.example.com
      - CORE_PEER_ADDRESS=peer1.org2.example.com:10051
      - CORE_PEER_LISTENADDRESS=0.0.0.0:10051
      - CORE_PEER_CHAINCODEADDRESS=peer1.org2.example.com:10052
      - CORE_PEER_CHAINCODELISTENADDRESS=0.0.0.0:10052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org2.example.com:10051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.example.com:9051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls:/etc/hyperledger/fabric/tls
        - peer1.org2.example.com:/var/hyperledgerproduction
    ports:
      - 10051:10051


```



### 2.1.5 拷贝文件到本地宿主机

完成以上步骤后检查生成的文件：

```bash
$ ls
businesschannel.tx  configtx.yaml  crypto-config  crypto-config.yaml  docker-compose-base.yaml  
docker-compose-cli.yaml  orderer.genesis.block  peer-base.yaml
```

由于我们当前生成的文件是在存储在 docker 中的，我们需要把它们拷贝到本地宿主机，拷贝命令为 docker cp [OPTIONS] [CONTAINER_ID]:[SRC_PATH] [DEST_PATH] ,其中 CONTAINER_ID 即为我们启动的 zhigui/z-ledger-tools 镜像的容器。例如以上生成的文件存储在 docker 中的目录为 /tmp/config ,在宿主机中用以下命令把这些文件拷贝到本地宿主机的 /tmp/config/ 目录：

```bash
$ docker cp acc4109fab78:/tmp/config/ /tmp/config/
```

完成上述工作后，就可以把 zhigui/z-ledger-tools 容器停止，因为后续我们还要采用docker-compose文件的形式来启动它。用以下命令停止容器：

```bash
$ docker stop CONTAINER
```



## 2.2 启动网络

如果你前面的配置文件都完成了，那么接下来就可以准备启动 Z-Ledger 网络了。

由于后续的操作需要用到链码的安装和初始化功能，所以我们需要先将需要的链码拷贝到当前工作目录。我们可以从github.com/hyperledger/fabric/examples/network/chaincode_example02 中拷贝 chaincode_example02 目录到当前目录。

在启动前请先检查当前目录是否拥有以下文件，如果没有请检查前面的步骤是否有什么疏漏的地方。

```bash
$ ls
businesschannel.tx		configtx.yaml			crypto-config.yaml		docker-compose-cli.yaml		peer-base.yaml
chaincode_example02		crypto-config			docker-compose-base.yaml	orderer.genesis.block
```

### 2.2.1 启动 Z-Ledger 节点服务

接着，运行命令启动 Z-Ledger 节点网络：

```bash
$ docker-compose -f docker-compose-cli.yaml up -d 
```

启动成功后，我们能看到如下的容器：

```bash
$ docker ps
CONTAINER ID        IMAGE                              COMMAND             CREATED             STATUS              PORTS                      NAMES
12a3f0c668a1        zhigui/z-ledger-tools:2.0     "/bin/bash"         24 seconds ago      Up 23 seconds                                  cli
839b9a361240        zhigui/z-ledger-peer:2.0      "peer node start"   25 seconds ago      Up 24 seconds       0.0.0.0:8051->8051/tcp     peer1.org1.example.com
31a397d57e79        zhigui/z-ledger-peer:2.0      "peer node start"   25 seconds ago      Up 24 seconds       0.0.0.0:7051->7051/tcp     peer0.org1.example.com
619410584b07        zhigui/z-ledger-peer:2.0      "peer node start"   25 seconds ago      Up 24 seconds       0.0.0.0:10051->10051/tcp   peer1.org2.example.com
6239b73db22e        zhigui/z-ledger-orderer:2.0   "orderer"           25 seconds ago      Up 24 seconds       0.0.0.0:7050->7050/tcp     orderer.example.com
2e8b902c1cf7        zhigui/z-ledger-peer:2.0      "peer node start"   25 seconds ago      Up 24 seconds       0.0.0.0:9051->9051/tcp     peer0.org2.example.com
```

实际上我们启动了一个客户端cli，一个 ordere r排序服务和四个 peer 节点。

### 2.2.2 启动 Z-Ledger CA 服务

Z-Ledger CA 就是Z-Ledger的证书颁发机构(CA)，CA的功能有：

- 身份信息注册, 或者连接到LDAP上创建用户信息
- 签发登记证书 (ECerts)
- 签发交易证书 (TCerts), 对在Z-Ledger区块链上的交易实现不可连接性和匿名性
- 证书更新和吊销

我们使用Docker Compose文件来启动CA服务器，下面docker-compose-ca.yaml文件实例：

```yaml
z-ledger-ca-server:
   image: zhigui/z-ledger-ca:2.0
   container_name: z-ledger-ca-server
   ports:
     - "7054:7054"
   environment:
     - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
     - CORE_PEER_TLS_ENABLED=false
   volumes:
     - "./z-ledger-ca:/etc/hyperledger/fabric-ca-server"
   command: sh -c 'fabric-ca-server start -b admin:adminpw'
```

接下来使用以下命令启动Z-Ledger-CA服务：

```bash
$ docker-compose -f docker-compose-ca.yaml up -d
```

启动成功后查看当前目录，可以看到增加了一个名为 z-ledger-ca 的文件夹，查看目录里面的内容：

```bash
$ ls
IssuerPublicKey			IssuerRevocationPublicKey	ca-cert.pem			fabric-ca-server-config.yaml	fabric-ca-server.db		msp
```

其中 ca-cert.pem 是服务器自签名的根证书，所有由该CA服务器发行的证书都需要该根证书进行签名。如果要指定该CA服务器为某个组织的CA服务器，则需要将 ca-cert.pem 替换为该组织的CA根证书。这将在后续的2.3.4第五章节进行展开讲解。

## 2.3 操作网络

启动完 Z-Ledger 节点服务和 Z-Ledger CA 服务后我们即可进行一系列交互操作。

### 2.3.1 创建通道

网络启动后，默认并不存在任何应用通道，需要手动创建应用通道，并让合作的 Peer 节点加入到通道中。下面在客户端进行相关操作。

我们通过以下命令进入 cli 客户端:

```bash
$ docker exec -it cli bash
root@12a3f0c668a1:/opt/gopath/src/github.com/hyperledger/fabric/peer# 
```

我们可以看到，我们已经进入到了docker容器中的 /opt/gopath/src/github.com/hyperledger/fabric/peer 目录，后续的操作我们都将在此目录进行。

输入以下命令查看可进行的操作：

```bash
$ peer
Usage:
  peer [command]

Available Commands:
  chaincode   Operate a chaincode: install|instantiate|invoke|package|query|signpackage|upgrade|list.
  channel     Operate a channel: create|fetch|join|list|update|signconfigtx|getinfo.
  help        Help about any command
  logging     Logging configuration: getlevel|setlevel|getlogspec|setlogspec|revertlevels.
  node        Operate a peer node: start|status|reset|rollback.
  version     Print fabric peer version.

Flags:
  -h, --help   help for peer

Use "peer [command] --help" for more information about a command.

```

接着我们就可以在客户端中进行相关的操作了，创建通道的命令如下：

```bash
$ CHANNEL_NAME=businesschannel
$ CORE_PEER_LOCALMSPID="Org1MSP" \
  peer channel create \
  -o orderer.example.com:7050 \
  -c $CHANNEL_NAME \
  -f ./channel-artifacts/businesschannel.tx
2020-06-19 09:48:55.272 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2020-06-19 09:48:55.311 UTC [cli.common] readBlock -> INFO 002 Received block: 0
```

创建通道成功后，会自动在本地生成该应用通道同名的初始区块 businesschannel.block 文件。只有拥有该文件才可以加入到创建的应用通道中。

### 2.3.2 加入通道

应用通道所包含组织的成员节点可以加入到通道中。在客户端使用管理员身份依次让组织 Org1 和 Org2 中所有节点都加入新的应用通道。操作需要指定所操作的 Peer 的地址，以及通道的初始区块。这里以操作 Org1 中的 peer0 节点为例： 

```bash
$ CORE_PEER_ADDRESS=peer0.org1.example.com:7051 \
peer channel join \
-b ${CHANNEL_NAME}.block 
2020-06-19 09:56:25.996 UTC [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
2020-06-19 09:56:26.061 UTC [channelCmd] executeJoin -> INFO 002 Successfully submitted proposal to join channel
```

### 2.3.3 测试链码

Peer 加入到应用通道后，可以执行链码相关操作，进行测试。链码在调用之前，必须先经过安装（Install）和实例化（Instantiate）两个步骤，部署到Peer 节点上。

通过如下命令，在客户端安装示例链码 chaincode_example02 到 Org1 的 Peer0 上：

```bash
$ CORE_PEER_ADDRESS=peer0.org1.example.com:7051 \
  peer chaincode install \
  -n mycc \
  -v 1.0 \
  -p "github.com/chaincode_example02/go/"
2020-06-19 13:53:28.875 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2020-06-19 13:53:28.875 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
2020-06-19 13:53:29.354 UTC [chaincodeCmd] install -> INFO 003 Installed remotely response:<status:200 payload:"OK" > 
```

通过如下命令将链码容器实例化，实例化操作的内容是把 a 初始化为 100，b 初始化为200。注意通过 -P 指定背书策略，此处 OR('Org1MSP.member','Org2MSP.member') 代表 Org1 或 Org2 的任意成员签名的交易即可调用该链码：

```bash
$ peer chaincode instantiate \
  -o orderer.example.com:7050 \
  -C ${CHANNEL_NAME} \
  -n mycc \
  -v 1.0 \
  -c '{"Args":["init","a","100","b","200"]}' \
  -P "OR ('Org1MSP.member','Org2MSP.member')" 
2020-06-19 14:50:57.443 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 001 Using default escc
2020-06-19 14:50:57.443 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 002 Using default vscc
```

实例化完成后，用户即可向网络中发起交易了。例如可以通过如下命令来调用链码： 

```bash
$ CORE_PEER_ADDRESS=peer0.org1.example.com:7051 \
  peer chaincode invoke \
  -o orderer.example.com:7050 \
  -C $CHANNEL_NAME \
  -n mycc \
  -c '{"Args":["invoke","a","b","10"]}' 
2020-06-19 14:55:47.197 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 001 Chaincode invoke successful. result: status:200
```

上步的操作是a 向 b 转账金额为 10 ，通过如下命令查询调用链码后的结果： 

```bash
$ CORE_PEER_LOCALMSPID="Org1MSP" \
  CORE_PEER_ADDRESS=peer0.org1.example.com:7051 \
  peer chaincode query \
  -n mycc \
  -C ${CHANNEL_NAME} \
  -c '{"Args":["query","a"]}'
90
```

结果返回为 90 ，符合预期。



### 2.3.4 注册证书

例如org1组织编写了一个客户端APP，那么该客户端如果要想访问Z-Ledger网络，就需要向org1的CA服务器申请相关证书。注册证书可以有两种方式，一种是使用z -ledger-ca 提供的客户端进行注册，另一种是使用 SDK 编写程序进行注册操作。下面分别介绍这两种方式。

当我们按照2.2.2的步骤启动好CA服务器后，需要拷贝crypto-config/peerOrganizations/org1.example.com/ca目录中的 ca.org1.example.com-cert.pem 、_sk结尾的文件和 crypto-config/peerOrganizations/org1.example.com/ ca.org1.example.com-cert.pem 目录中的 tlsca.org1.example.com-cert.pem 文件到 z-ledger-ca 目录中。并且将 ca.org1.example.com-cert.pem 重命名为 ca-cert.pem 替换掉之前生成的 ca-cert.pem，将 tlsca.org1.example.com-cert.pem 文件重命名为 tls-cert.pem，\_sk结尾的文件重命名为ca-key.pem。

其中 ca-cert.pem 为 CA 服务器的的根证书，CA 服务器此后生成的证书都是由该根证书签发的，由此形成证书链。ca-key.pem 用来对新生成证书进行签名。tls-cert.pem 为进行 tls 连接时需要的证书。

**使用客户端注册证书**

要启动 z-ledger-ca客户端和服务器进行交互，需要编写z-ledger-ca客户端的配置文件，配置文件和 CA 服务器类似，参考配置如下：

```yaml

# URL of the Fabric-ca-server (default: http://localhost:7054)
url: http://localhost:7054

# Membership Service Provider (MSP) directory
# This is useful when the client is used to enroll a peer or orderer, so
# that the enrollment artifacts are stored in the format expected by MSP.
mspdir: msp

#############################################################################
#    TLS section for secure socket connection
#
#  certfiles - PEM-encoded list of trusted root certificate files
#  client:
#    certfile - PEM-encoded certificate file for when client authentication
#    is enabled on server
#    keyfile - PEM-encoded key file for when client authentication
#    is enabled on server
#############################################################################
tls:
  # TLS section for secure socket connection
  enabled: false
  #certfiles:
  certfiles: /etc/hyperledger/fabric-ca-server/ca-cert.pem
  client:
    certfile: /etc/hyperledger/fabric-ca-server/tls-cert.pem
    #certfile:
    keyfile: 

#############################################################################
#  Certificate Signing Request section for generating the CSR for an
#  enrollment certificate (ECert)
#
#  cn - Used by CAs to determine which domain the certificate is to be generated for
#
#  serialnumber - The serialnumber field, if specified, becomes part of the issued
#     certificate's DN (Distinguished Name).  For example, one use case for this is
#     a company with its own CA (Certificate Authority) which issues certificates
#     to its employees and wants to include the employee's serial number in the DN
#     of its issued certificates.
#     WARNING: The serialnumber field should not be confused with the certificate's
#     serial number which is set by the CA but is not a component of the
#     certificate's DN.
#
#  names -  A list of name objects. Each name object should contain at least one
#    "C", "L", "O", or "ST" value (or any combination of these) where these
#    are abbreviations for the following:
#        "C": country
#        "L": locality or municipality (such as city or town name)
#        "O": organization
#        "OU": organizational unit, such as the department responsible for owning the key;
#         it can also be used for a "Doing Business As" (DBS) name
#        "ST": the state or province
#
#    Note that the "OU" or organizational units of an ECert are always set according
#    to the values of the identities type and affiliation. OUs are calculated for an enroll
#    as OU=<type>, OU=<affiliationRoot>, ..., OU=<affiliationLeaf>. For example, an identity
#    of type "client" with an affiliation of "org1.dept2.team3" would have the following
#    organizational units: OU=client, OU=org1, OU=dept2, OU=team3
#
#  hosts - A list of host names for which the certificate should be valid
#
#############################################################################
csr:
  cn: admin
  keyrequest:
    algo: ecdsa
    size: 256
  serialnumber:
  names:
    - C: CH
      ST: Bei Jing
      L:
      O: ZhiGui
      OU: fabric
  hosts:
    - wanggang

#############################################################################
#  Registration section used to register a new identity with fabric-ca server
#
#  name - Unique name of the identity
#  type - Type of identity being registered (e.g. 'peer, app, user')
#  affiliation - The identity's affiliation
#  maxenrollments - The maximum number of times the secret can be reused to enroll.
#                   Specially, -1 means unlimited; 0 means to use CA's max enrollment
#                   value.
#  attributes - List of name/value pairs of attribute for identity
#############################################################################
id:
  name:
  type:
  affiliation:
  maxenrollments: 0
  attributes:
   # - name:
   #   value:

#############################################################################
#  Enrollment section used to enroll an identity with fabric-ca server
#
#  profile - Name of the signing profile to use in issuing the certificate
#  label - Label to use in HSM operations
#############################################################################
enrollment:
  profile:
  label:

#############################################################################
# Name of the CA to connect to within the fabric-ca server
#############################################################################
caname:

#############################################################################
# BCCSP (BlockChain Crypto Service Provider) section allows to select which
# crypto implementation library to use
#############################################################################
bccsp:
    default: SW
    sw:
        algorithm: ECDSA
        hash: SHA2
        security: 256
        filekeystore:
            # The directory used for the software file-based keystore
            keystore: msp/keystore

```

将上述文件命名为 fabric-ca-client-config.yaml ，同样放在本地的 z-ledger-ca 目录中。完成之后，进入 z-ledger-ca 容器。

```bash
$ docker exec -it z-ledger-ca-server bash
```

进入 Docker 容器后在容器中执行以下命令查看客户端相关命令:

```bash
$ cd /etc/hyperledger/fabric-ca-server/
$ fabric-ca-client
  Hyperledger Fabric Certificate Authority Client

  Usage:
    fabric-ca-client [command]

  Available Commands:
    affiliation Manage affiliations
    certificate Manage certificates
    enroll      Enroll an identity
    gencrl      Generate a CRL
    gencsr      Generate a CSR
    getcainfo   Get CA certificate chain and Idemix public key
    identity    Manage identities
    reenroll    Reenroll an identity
    register    Register an identity
    revoke      Revoke an identity
    version     Prints Fabric CA Client version

```

首先登陆一个服务器用户才能用该用户来进行接下来的注册操作，下面登陆 admin 用户。

```bash
$ fabric-ca-client enroll -u http://admin:adminpw@localhost:7054
# 返回信息
2020/07/08 02:09:13 [INFO] generating key: &{A:ecdsa S:256}
2020/07/08 02:09:13 [INFO] encoded CSR
2020/07/08 02:09:13 [INFO] Stored client certificate at /etc/hyperledger/fabric-ca-server/msp/signcerts/cert.pem
2020/07/08 02:09:13 [INFO] Stored root CA certificate at /etc/hyperledger/fabric-ca-server/msp/cacerts/localhost-7054.pem
2020/07/08 02:09:13 [INFO] Stored Issuer public key at /etc/hyperledger/fabric-ca-server/msp/IssuerPublicKey
2020/07/08 02:09:13 [INFO] Stored Issuer revocation public key at /etc/hyperledger/fabric-ca-server/msp/IssuerRevocationPublicKey
```

接着注册一个名为 user , 密码为 user-pw，所属部门为 org1.department1 的用户。

```bash
$ fabric-ca-client register \
--id.name user \
--id.secret user-pw \
--id.affiliation org1.department1 \
-u http://admin:adminpwd@localhost:7054
# 返回信息
2020/07/08 02:14:13 [INFO] Configuration file location: /etc/hyperledger/fabric-ca-server/fabric-ca-client-config.yaml
Password: user-pw
```

上一步的 register 操作仅仅是注册了用户的信息，接下来的 enroll 操作获取颁发的证书。

```bash
$ fabric-ca-client enroll -u http://user:user-pw@localhost:7054
# 返回信息
2020/07/08 02:22:36 [INFO] generating key: &{A:ecdsa S:256}
2020/07/08 02:22:36 [INFO] encoded CSR
2020/07/08 02:22:36 [INFO] Stored client certificate at /etc/hyperledger/fabric-ca-server/msp/signcerts/cert.pem
2020/07/08 02:22:36 [INFO] Stored root CA certificate at /etc/hyperledger/fabric-ca-server/msp/cacerts/localhost-7054.pem
2020/07/08 02:22:36 [INFO] Stored Issuer public key at /etc/hyperledger/fabric-ca-server/msp/IssuerPublicKey
2020/07/08 02:22:36 [INFO] Stored Issuer revocation public key at /etc/hyperledger/fabric-ca-server/msp/IssuerRevocationPublicKey
```

后续我们就可以使用生成的证书来访问Z-Ledger网络了。

**使用 SDK 注册证书**

CA服务启动成功后也可以使用SDK编写程序调用CA服务器来注册新节点所需要的证书。使用SDK的例子可从https://github.com/maluning/z-ledger-sdk-go-sample进行下载。

```bash
$ cd $GOPATH/src/github.com/hyperledger/
$ git clone https://github.com/maluning/z-ledger-sdk-go-sample.git
```

进入下载好的代码目录，你需要将 org1-config.yaml 中的有关 crypto-config 所在目录URL替换成自己本地实际的URL。执行以下命令，启动连接CA的HTTP服务：

```bash
$ go build
$ ./z-ledger-sdk-go-sample ca
```

在浏览器中输入以下URL登陆admin用户：

```
访问：localhost:9091/msp/enroll/admin/adminpw
// 返回结果如下
{
    "result": "enroll admin successful"
}
```

然后就可以分别访问以下地址注册一个名为user的用户：

```
访问：localhost:9091/msp/enroll/user/userpw
// 返回结果如下
{
    "result": "register user successful",
    "secret": "userpw"
}

访问：localhost:9091/msp/enroll/user/userpw
// 返回结果如下
{
    "result": "enroll user successful"
}
```

经过以上步骤后就将用户注册成功了，查看/tmp/examplestore目录我们将看到之前生成的证书：

```bash
$ ls
admin@Org1MSP-cert.pem	user@Org1MSP-cert.pem
```

# 三.使用链上代码

链上代码（Chaincode），或简称链码，一般指的是用户编写的应用代码。链码被部署在 Z-Ledger 网络节点上，运行在隔离沙盒（目前为 Docker 容器）中，并通过gRPC 协议来与相应的 Peer 节点进行交互，以操作分布式账本中的数据。

## 3.1 链码操作命令

用户可以通过命令行方式操作链码，支持的链码子命令包括 install、instantiate、invoke、query、upgrade、package、signpackage 等。后面将以 Z-Ledger 项目中自带的 Go 语言 example02 链码（路径在examples/chaincode/go/chaincode_example02）为例进行相关命令讲解。

## 3.2 安装链码

install 命令将链码的源码和环境等内容封装为一个链码安装打包文件（Chaincode Install Package，CIP），并传输到背书节点。背书节点解析后一般会保存在 $CORE_PEER_FILESYSTEMPATH/chaincodes/ 目录下。安装链码只需要跟 Peer 打交道。打包文件以 name.version 命名。

例如采用如下命令会部署 mycc.1.0 的打包部署文件到背书节点：

```bash
# peer chaincode install \
  -n mycc \
  -v 1.0 \
  -p "github.com/chaincode_example02/go/"
```

## 3.3 实例化链码

instantiate 命令通过构造生命周期管理系统链码（Lifecycle System Chaincode，LSCC）的交易，将安装过的链码在指定通道上进行实例化调用，在节点上创建容器启动，并执行初始化操作。实例化链码需要同时跟 Peer 和 Orderer 打交道。

执行 instantiate 命令的用户身份必须满足实例化的策略，并且在所指定的通道上拥有写（Write）权限。在 instantiate 命令中可以通过“-P”参数指定链码的背书策略（Endorsement Policy），不满足背书策略的链码调用将在 Commit 阶段被作废。

例如，如下命令会启动 mycc.1.0 链码，会将参数 '{"Args":["init","a","100","b","200"]}' 传入链码中的 Init() 方法执行。命令会生成一笔交易，因此需指定排序节点地址。 

```bash
# peer chaincode instantiate \
  -o orderer.example.com:7050 \
  -C ${CHANNEL_NAME} \
  -n mycc \
  -v 1.0 \
  -c '{"Args":["init","a","100","b","200"]}' \
  -P "OR ('Org1MSP.member','Org2MSP.member')" 
```

## 3.4 调用链码

通过 invoke 命令可以调用运行中的链码的方法。“-c” 参数指定的函数名和参数会被传入到链码的 Invoke() 方法进行处理。调用链码操作需要同时跟 Peer 和 Orderer 打交道。例如，对部署成功的链码执行调用操作，由 a 向 b 转账 10 元。

```bash
# peer chaincode invoke \
  -o orderer.example.com:7050 \
  -C $CHANNEL_NAME \
  -n mycc \
  -c '{"Args":["invoke","a","b","10"]}' 
```

## 3.5 查询链码

查询链码可以通过 query 命令。query 命令的执行过程与 invoke 命令类似，实际上同样是将 -c 指定的命令参数发送给了链码中的 Invoke() 方法执行。与 invoke 操作的区别在于，query 操作只能查询 Peer 上账本状态，不生成交易，也不需要与 Orderer 打交道。

例如，执行如下命令会调用最新版本的 mycc 链码，将参数 '{"Args":["query","a"]}' 传入链码中的 Invoke() 方法执行，并返回查询结果。

```bash
# peer chaincode query \
  -n mycc \
  -C ${CHANNEL_NAME} \
  -c '{"Args":["query","a"]}'
Query Result: 100
```

## 3.6 升级链码

当需要修复链码漏洞或进行功能拓展时，可以对链码进行升级，部署新版本的链码。Z-Ledger 支持在保留现有状态的前提下对链码进行升级。

假设某通道上正在运行中的链码为 mycc，版本为 1.0，可以通过如下步骤进行升级操作。首先，安装新版本的链码，打包到 Peer 节点： 

```bash
# peer chaincode install \
  -n mycc \
  -v 1.1 \
  -p "github.com/chaincode_example02/go/"
```

运行以下 upgrade 命令升级指定通道上的链码，需要指定相同的链码名称 mycc： 

```bash
# peer chaincode upgrade \
  -C ${CHANNEL_NAME} \
  -n mycc \
  -v 1.1 \
  -c '{"Args":["re-init","c","60"]}' 
```

这一命令会在通道 上实例化新版本链码 mycc.1.1 并启动一个新容器。运行在其他通道上的旧版本的链码将不受影响。升级操作跟实例化操作十分类似，唯一区别在于不改变实例化的策略。这就保证了只有拥有实例化权限的用户才能进行升级操作。

## 3.7 打包链码和签名

通过将链码相关的数据进行封装，可以实现对其进行打包和签名操作。

打包命令支持三个特定参数：

- -s, --cc-package：表示创建完整打包格式，而不是仅打包 ChaincodeDeploymentSpec 结构。 

-  -S, --sign：对打包的文件使用本地的 MSP（core.yaml 中的 localMspid 指定）进行签名。

-  -i --instantiate-policy string：指定实例化策略。可选参数。

例如，通过如下命令创建一个本地的打包文件 ccpack.out：

```bash
# peer chaincode package \
-n test_cc \
-p github.com/chaincode_example02/go/ \
-v 1.0 \
-s \
-S \
-i "AND('Org1.admin')" \
ccpack.out
```

打包后的文件，也可以直接用于 install 操作，如：

```bash
# peer chaincode install ccpack.out
```

签名命令则对一个打包文件进行签名操作（添加当前 MSP 签名到签名列表中）。

```bash
# peer chaincode signpackage ccpack.out signedccpack.out
```



# 四.使用多通道

## 4.1 通道操作命令

命令行下 peer channel 命令支持包括 create、fetch、join、list、update 等子命令。各个命令的功能如下所示：

-  create：创建一个新的应用通道。

- join：将本 Peer 节点加入到某个应用通道中。 

-  list：列出本 Peer 已经加入的所有的应用通道。

- fetch：从 Ordering 服务获取指定应用通道的配置区块。 

-  update：更新通道的配置信息，如锚节点配置。

可以通过 peer channel <subcommand> --help 来查看具体的命令使用说明。

## 4.2 创建通道

create 子命令由拥有创建通道权限的组织的管理员身份来调用，在指定的 Ordering 服务上

创建新的应用通道，需要提供 Ordering 服务地址。

一般情况下，通过提前创建的通道配置交易文件来指定配置信息。如果不指定通道配置文件，则默认采用 SampleConsortium 配置和本地的 MSP 组织来构造配置交易结构。

例如，下面命令利用事先创建的配置交易文件 channel.tx 来创建新的应用通道businesschannel。

```bash
# peer channel create \
  -o orderer.example.com:7050 \
  -c $CHANNEL_NAME \
  -f ./channel-artifacts/businesschannel.tx
```

## 4.3 加入通道

join 子命令会让指定的 Peer 节点加入到指定的应用通道。需要提前拥有所加入应用通道的初始区块文件，并且只有属于通道的某个组织的管理员身份可以成功执行该操作。加入通道命令主要通过调用 Peer 的配置系统链码进行处理。

例如，通过如下命令将本地 Peer 加入到应用通道 businesschannel 中。

```bash
# peer channel join \
	-b ${CHANNEL_NAME}.block 
	-o orderer:7050

Peer joined the channel!
```

## 4.4 列出所加入的通道

list 子命令会列出指定的 Peer 节点已经加入的所有应用通道的列表。加入通道命令也是主要通过调用 Peer 的配置系统链码进行处理。

例如通过如下命令，可以列出本地 Peer 已经加入的所有应用通道。

```bash
# peer channel list
Channels peers has joined to:
businesschannel
businesschannel2
```

## 4.5 获取某区块

fetch 子命令会向 Ordering 服务进行查询，获取到指定通道的指定区块。并将收到的区块写入到本地的文件（默认为 chainID_序号.block）。

命令格式为 peer channel fetch <newest|oldest|config|(number)> [outputfile] [flags]。

例如通过如下命令，可以获取到已存在的 businesschannel 应用通道的初始区块，并保存到本地的 businesschannel.block 文件。

```bash
# peer channel fetch oldest businesschannel.block \
-c businesschannel \
-o orderer:7050
```

## 4.6 更新通道配置

update 子命令的执行过程与 create 命令类似，会向 Ordering 服务发起更新配置交易请求。该命令执行也需要提前创建的通道更新配置交易文件来指定配置信息。

例如，通过如下操作来更新通道中的锚节点配置，首先利用 configtxgen 来创建锚节点配置更新文件，之后使用该更新文件对通道进行配置更新操作。

```bash
# configtxgen \
-profile TwoOrgsChannel \
-outputAnchorPeersUpdate ./update_anchors.tx \
-channelID businesschannel \
-asOrg Org1MSP

$ peer channel update \
-c businesschannel \
-o orderer:7050 \
-f ./update_anchors.tx
```



# 五.使用 CA 服务

## 5.1 CA客户端操作命令

进入 z-ledger-ca-server 容器后， 查看 z-ledger-ca 客户端支持的命令。

```bash
$ fabric-ca-client
Hyperledger Fabric Certificate Authority Client

Usage:
  fabric-ca-client [command]

Available Commands:
  affiliation Manage affiliations
  certificate Manage certificates
  enroll      Enroll an identity
  gencrl      Generate a CRL
  gencsr      Generate a CSR
  getcainfo   Get CA certificate chain and Idemix public key
  identity    Manage identities
  reenroll    Reenroll an identity
  register    Register an identity
  revoke      Revoke an identity
  version     Prints Fabric CA Client version

```

主要功能如下： 

- enroll：登录获取 ECert； 
- getcacert：获取 CA 服务的证书链； 
- reenroll：再次登录； 
-  register：注册用户实体； 
-  revoke：吊销签发的实体证书。

## 5.2 enroll 命令

命令格式为 fabric-ca-client enroll -u http://user:userpw@serverAddr:serverPort。该命令会向服务端申请签发 ECert 证书。

如采用下面命令获取证书文件保存到本地：

```bash
$ fabric-ca-client enroll -u http://admin:adminpw@localhost:7054
# 返回信息
2020/07/08 02:09:13 [INFO] generating key: &{A:ecdsa S:256}
2020/07/08 02:09:13 [INFO] encoded CSR
2020/07/08 02:09:13 [INFO] Stored client certificate at /etc/hyperledger/fabric-ca-server/msp/signcerts/cert.pem
2020/07/08 02:09:13 [INFO] Stored root CA certificate at /etc/hyperledger/fabric-ca-server/msp/cacerts/localhost-7054.pem
2020/07/08 02:09:13 [INFO] Stored Issuer public key at /etc/hyperledger/fabric-ca-server/msp/IssuerPublicKey
2020/07/08 02:09:13 [INFO] Stored Issuer revocation public key at /etc/hyperledger/fabric-ca-server/msp/IssuerRevocationPublicKey
```

该命令会在默认的主配置目录下创建 msp 目录，其中 cacerts 目录下存放有服务端的证书，signcerts 目录下存放服务端签发的代表客户端身份的证书，keystore 目录下是对应客户端签名证书的私钥文件。

## 5.3 getcacert 命令

命令格式为 fabric-ca-client getcacert -u http://serverAddr:serverPort -M <MSP-directory>[flags]。该命令会向服务端申请根证书信息。

例如采用下面命令获取服务端证书文件保存到本地主配置目录的 msp/cacerts 路径下： 

```bash
$ fabric-ca-client getcacert -u http://admin:adminpw@localhost:7054
# 返回信息
2020/07/08 02:56:33 [INFO] Configuration file location: /etc/hyperledger/fabric-ca-server/fabric-ca-client-config.yaml
2020/07/08 02:56:33 [INFO] Stored root CA certificate at /etc/hyperledger/fabric-ca-server/msp/cacerts/localhost-7054.pem
2020/07/08 02:56:33 [INFO] Stored Issuer public key at /etc/hyperledger/fabric-ca-server/msp/IssuerPublicKey
2020/07/08 02:56:33 [INFO] Stored Issuer revocation public key at /etc/hyperledger/fabric-ca-server/msp/IssuerRevocationPublicKey
```

## 5.4 reenroll 命令

命令格式为 fabric-ca-client reenroll [flags]。会利用本地配置信息，再次执行 enroll 过程，生成新的签名证书材料。

执行过程如下所示，与 enroll 过程类似，获取新的证书文件：

```bash
$ fabric-ca-client reenroll
# 返回信息
2020/07/08 02:58:07 [INFO] Configuration file location: /etc/hyperledger/fabric-ca-server/fabric-ca-client-config.yaml
2020/07/08 02:58:07 [INFO] generating key: &{A:ecdsa S:256}
2020/07/08 02:58:07 [INFO] encoded CSR
2020/07/08 02:58:07 [INFO] Stored client certificate at /etc/hyperledger/fabric-ca-server/msp/signcerts/cert.pem
2020/07/08 02:58:07 [INFO] Stored root CA certificate at /etc/hyperledger/fabric-ca-server/msp/cacerts/localhost-7054.pem
```

## 5.5 register 命令

命令格式为 fabric-ca-client register [flags]。注册新的用户实体身份。

执行注册新用户实体的客户端必须已经通过了登记认证，并且拥有足够的权限（所注册用户的 hf.Registrar.Roles 和 affiliation 都不能超出调用者属性）来进行注册。

例如通过如下命令来注册新的用户 new_user，注册成功后系统会返回用来登记的密码：

```bash
$ fabric-ca-client register \
--id.name new_user \
--id.type user \
--id.affiliation org1.department1 \
--id.attr hf.Revoker=true
# 返回信息
2020/07/08 03:09:51 [INFO] Configuration file location: /etc/hyperledger/fabric-ca-server/fabric-ca-client-config.yaml
Password: yIUmotUILLAN
```

## 5.6 revoke 命令

命令格式为 fabric-ca-client revoke [flags]。

revoke 命令会吊销指定的证书或者指定实体相关的所有证书。执行 revoke 命令的客户端的身份必须拥有足够的权限（hf.Revoker 为 true，并且被吊销者机构不能超出吊销者机构的范围）。

例如通过如下命令吊销用户 new_user，并通过 -r 指定原因： 

```bash
$  fabric-ca-client revoke -e "new_user" -r "affiliationchange"
# 返回信息
2020/07/08 03:17:33 [INFO] Configuration file location: /etc/hyperledger/fabric-ca-server/fabric-ca-client-config.yaml
2020/07/08 03:17:33 [INFO] Sucessfully revoked certificates: [{Serial:5dfe6b5e89580e94a6ca996bf44b49b07bc2fd72 AKI:5de4351d98e95bd4401e994467e6ae8c69045da91e04d5f0aa12445e0d4dd0ea}]
```

