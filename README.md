# Blockchain Solution with Hyperledger Fabric + Hyperledger Explorer on Kubernetes


**Maintainers:** [feitnomore](https://github.com/feitnomore/)

This is a simple guide to help you implement a complete Blockchain solution using [Hyperledger Fabric v1.3](https://hyperledger-fabric.readthedocs.io/en/release-1.3/whatsnew.html) with [Hyperledger Explorer v0.3.7](https://www.hyperledger.org/projects/explorer) on top of a [Kubernetes](https://kubernetes.io) platform.
This solution uses also [CouchDB](http://couchdb.apache.org/) as peer's backend, [Apache Kafka](https://kafka.apache.org/) topics for the orderers and a NFS Server *(Network file system)* to share data between the components.

*WARNING:* Use it at your own risk.

## BACKGROUND

A few weeks back, I've decided to take a look at Hyperledger Fabric solution to Blockchain, as it seems to be a technology that has been seeing an increase use and also is supported by giant tech companies like IBM and Oracle for example.
When I started looking at it, I've found lots of scripts like `start.sh`, `stop.sh`, `byfn.sh` and `eyfn.sh`. For me those seems like "magic", and everyone that I've talked to, stated that I should use those.
While using those scripts made me start fast, I had lots of trouble figuring out what was going on *behind the scenes* and also had a really hard time trying to customize the environment or run anything different from those samples.
At that point I've decided to start digging and started building a complete Blockchain environment, step-by-step, in order to see the details of how it works and how it can be achieved. This github repository is the result of my studies.

## INTRODUCTION

We're going to build a complete Hyperledger Fabric v1.3 environment with CA, Orderer and 4 Organizations. In order to achieve scalability and high availability on the Orderer we're going to be using Kafka. Each Organization will have 2 peers, and each peer will have it's own CouchDB instance. We're also going to deploy Hyperledger Explorer v0.3.7 with its PostgreSQL database as well.

## ARCHITECTURE

### Infrastructure view

For this environment we're going to be using a 3-node Kubernetes cluster, a 3-node Apache Zookeeper cluster (for Kafka), a 4-node Apache Kafka cluster and a NFS server. All the machines are going to be in the same network.
For Kubernetes cluster we'll have the following machines:
```sh
kubenode01.local.parisi.biz
kubenode02.local.parisi.biz
kubenode03.local.parisi.biz
```
*Note: This is a home Kubernetes environment however most of what is covered here should apply to any cloud provider that provides Kubernetes compatible services*  

For Apache Zookeeper we'll have the following machines:
```sh
zookeeper1.local.parisi.biz
zookeeper2.local.parisi.biz
zookeeper3.local.parisi.biz
```
*Note: Zookeeper is needed by Apache Kafka*  
*Note: Apache Kafka should be 1.0 for Hyperledger compatibility.*  
*Note: Check [this link](https://dzone.com/articles/how-to-setup-kafka-cluster) for a quick guide on Kafka/Zookeeper cluster*  

For Apache Kafka we'll have the following machines:
```sh
kafka1.local.parisi.biz
kafka2.local.parisi.biz
kafka3.local.parisi.biz
kafka4.local.parisi.biz
```
*Note 1: We're using Kafka 1.0 version for Hyperledger compatibility*  
*Note 2: Check [this link](https://dzone.com/articles/how-to-setup-kafka-cluster) for a quick guide on Kafka/Zookeeper cluster* 

For the NFS Server we'll have:
```sh
storage.local.parisi.biz
```
*Note: Check [this link](https://www.howtoforge.com/nfs-server-and-client-on-centos-7) for a quick guide on NFS Server setup*  

The image below represents the environment infrastructure:  

![slide1.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide1.jpg)


*Note: It's important to have all the environment with the time in sync as we're dealing with transactions and shared storage. Please make sure you have all the time in sync. I encourage you to use NTP on your servers. On my environment I have `ntpdate` running in a cron job.* 

### Fabric Logical view

This environment will have a CA and a Orderer as Kubernetes deployments:
```sh
blockchain-ca
blockchain-orderer
```

We'll also have 4 organizations, with each organization having 2 peers, organized in the following deployments:
```sh
blockchain-org1peer1
blockchain-org1peer2
blockchain-org2peer1
blockchain-org2peer2
blockchain-org3peer1
blockchain-org3peer2
blockchain-org4peer1
blockchain-org4peer2
```

The image below represents this logical view:  
 
![slide2.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide2.jpg)

### Explorer Logical view
We're going to have Hyperledger Explorer as a WebUI for our environment. Hyperledger Explorer will run in 2 deployments as below:
```sh
blockchain-explorer-db
blockchain-explorer-app
```

The image below represents this logical view:  
 
![slide3.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide3.jpg)

### Detailed view
Hyperledger Fabric Orderer will connect itself to the Kafka servers as image below:  
  
![slide4.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide4.jpg)

Each Hyperledger Fabric Peer will have it's own CouchDB instance running as a sidecar and will connect to our NFS shared storage:  

![slide5.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide5.jpg)

*Note: Although its not depicted above, CA, Orderer and Explorer deployments will also have access to the NFS shared storage as they need the artifacts that we're going to store there.*

## IMPLEMENTATION

### Step 1: Checking environment

First let's make sure we have Kubernetes environment up & running:
```sh
kubectl get nodes
```

### Step 2: Setting up shared storage

Now, assuming the NFS server is up & running and with the correct permissions, we're going to create our `PersistentVolume`. First lets create the file `kubernetes/fabric-pv.yaml` like the example below:
```yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: fabric-pv
  labels:
    type: local
    name: fabricfiles
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /nfs/fabric
    server: storage.local.parisi.biz
    readOnly: false
```

*Note: NFS Server is running on `storage.local.parisi.biz` and the shared filesystem is `/nfs/fabric`. We're using `fabricfiles` as the name for this PersistentVolume.*  

Now let's apply the above configuration:
```sh 
kubectl apply -f kubernetes/fabric-pv.yaml
```

After that we'll need to create a `PersistentVolumeClaim`. To do that, we'll create file `kubernetes/fabric-pvc.yaml` as below:
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: fabric-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
  selector:
    matchLabels:
      name: fabricfiles
```
*Note: We're using our previously created `fabricfiles` as the selector here.*  

Now let's apply the above configuration:
```sh
kubectl apply -f kubernetes/fabric-pvc.yaml
```

### Step 3: Launching a Fabric Tools helper pod

In order to perform some operations on the environment like file management, peer configuration and artifact generation, we'll need a helper `Pod` running `fabric-tools`. For that we'll create file `kubernetes/fabric-tools.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: fabric-tools
spec:
  volumes:
  - name: fabricfiles
    persistentVolumeClaim:
      claimName: fabric-pvc
  - name: dockersocket
    hostPath:
      path: /var/run/docker.sock
  containers:
    - name: fabrictools
      image: hyperledger/fabric-tools:amd64-1.3.0
      imagePullPolicy: Always
      command: ["sh", "-c", "sleep 48h"]
      env:
      - name: TZ
        value: "America/Sao_Paulo"
      - name: FABRIC_CFG_PATH
        value: "/fabric"
      volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
```

After creating the file, let's apply it to our kubernetes cluster: 
```sh
kubectl apply -f kubernetes/fabric-tools.yaml
```

Make sure the fabric-tools pod is running before we continue:
```sh
kubectl get pods
```

Now, assuming `fabric-tools` pod is running, lets create a config directory on our shared filesystem to hold our files:
```sh
kubectl exec -it fabric-tools -- mkdir /fabric/config
```

### Step 4: Loading the config files into the storage

1 - Configtx  
Now we're going to create the file `config/configtx.yaml` with our network configuration, like the example below:
```yaml
---
Organizations:

    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        MSPDir: crypto-config/ordererOrganizations/example.com/msp
        AdminPrincipal: Role.MEMBER

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.example.com/msp
        AdminPrincipal: Role.MEMBER
        AnchorPeers:
            - Host: blockchain-org1peer1
              Port: 30110
            - Host: blockchain-org1peer2
              Port: 30110

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.example.com/msp
        AdminPrincipal: Role.MEMBER
        AnchorPeers:
            - Host: blockchain-org2peer1
              Port: 30110
            - Host: blockchain-org2peer2
              Port: 30110

    - &Org3
        Name: Org3MSP
        ID: Org3MSP
        MSPDir: crypto-config/peerOrganizations/org3.example.com/msp
        AdminPrincipal: Role.MEMBER
        AnchorPeers:
            - Host: blockchain-org3peer1
              Port: 30110
            - Host: blockchain-org3peer2
              Port: 30110

    - &Org4
        Name: Org4MSP
        ID: Org4MSP
        MSPDir: crypto-config/peerOrganizations/org4.example.com/msp
        AdminPrincipal: Role.MEMBER
        AnchorPeers:
            - Host: blockchain-org4peer1
              Port: 30110
            - Host: blockchain-org4peer2
              Port: 30110

Orderer: &OrdererDefaults

    OrdererType: kafka
    Addresses:
        - blockchain-orderer:31010

    BatchTimeout: 1s
    BatchSize:
        MaxMessageCount: 50
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB

    Kafka:
        Brokers:
            - kafka1.local.parisi.biz:9092
            - kafka2.local.parisi.biz:9092
            - kafka3.local.parisi.biz:9092
            - kafka4.local.parisi.biz:9092

    Organizations:

Application: &ApplicationDefaults

    Organizations:

Profiles:

    FourOrgsOrdererGenesis:
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
                    - *Org3
                    - *Org4
    FourOrgsChannel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2
                - *Org3
                - *Org4
```
*Note: The file reflects the topology discussed on the architecture presented before.*  
*Note: Pay attention to the Kafka brokers URLs.*  
*Note: Its important to have Anchor Peers configuration here as it impacts Hyperledger Fabric Service Discovery.*  
*Note: BatchTimeout and BatchSize impacts directly in the performance of your environment in terms of quantity of transactions that are processed.*  

Now let's copy the file we just created to our shared filesystem:
```sh 
kubectl cp config/configtx.yaml fabric-tools:/fabric/config/
```

2 - Crypto-config  
Now lets create the file `config/crypto-config.yaml` like below:  
```yaml
OrdererOrgs:
  - Name: Orderer
    Domain: example.com
    Specs:
      - Hostname: orderer
PeerOrgs:
  - Name: Org1
    Domain: org1.example.com
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.example.com
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org3
    Domain: org3.example.com
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org4
    Domain: org4.example.com
    Template:
      Count: 2
    Users:
      Count: 1
```

Let's copy the file to our shared filesystem: 
```sh
kubectl cp config/crypto-config.yaml fabric-tools:/fabric/config/
```

3 - Chaincode  
It's time to copy our example chaincode to the shared filesystem. In this case we'll be using balance-transfer example:
```sh
kubectl cp config/chaincode/ fabric-tools:/fabric/config/
```

### Step 5: Creating the necessary artifacts

1 - cryptogen  
Time to generate our crypto material:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cryptogen generate --config /fabric/config/crypto-config.yaml
exit
```

Now we're going to copy our files to the correct path and rename the key files:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp -r crypto-config /fabric/
for file in $(find /fabric/ -iname *_sk); do echo $file; dir=$(dirname $file); mv ${dir}/*_sk ${dir}/key.pem; done
exit
```


2 - configtxgen  
Now we're going to copy the artifacts to the correct path and generate the genesis block:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp /fabric/config/configtx.yaml /fabric/
cd /fabric
configtxgen -profile FourOrgsOrdererGenesis -outputBlock genesis.block
exit
``` 

3 - Anchor Peers  
Lets create the Anchor Peers configuration files using configtxgen: 
```sh
kubectl exec -it fabric-tools -- /bin/bash
cd /fabric
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./Org1MSPanchors.tx -channelID channel1 -asOrg Org1MSP
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./Org2MSPanchors.tx -channelID channel1 -asOrg Org2MSP
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./Org3MSPanchors.tx -channelID channel1 -asOrg Org3MSP
configtxgen -profile FourOrgsChannel -outputAnchorPeersUpdate ./Org4MSPanchors.tx -channelID channel1 -asOrg Org4MSP
exit
```
*Note: The generated files will be used later to update channel configuration with the respective Anchor Peers. This step is important for Hyperledger Fabric Service Discovery to work properly.*   

4 - Fix Permissions  
We need to fix the files permissions on our shared filesystem now:
```sh
kubectl exec -it fabric-tools -- /bin/bash
chmod a+rx /fabric/* -R
exit
```


### Step 6: Setting up Fabric CA

Create the `kubernetes/blockchain-ca_deploy.yaml` file with the following `Deployment` description:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-ca
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: ca
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc

      containers:
      - name: ca
        image: hyperledger/fabric-ca:amd64-1.3.0
        command: ["sh", "-c", "fabric-ca-server start -b admin:adminpw -d"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: FABRIC_CA_SERVER_CA_NAME
          value: "CA1"
        - name: FABRIC_CA_SERVER_CA_CERTFILE
          value: /fabric/crypto-config/peerOrganizations/org1.example.com/ca/ca.org1.example.com-cert.pem
        - name: FABRIC_CA_SERVER_CA_KEYFILE
          value: /fabric/crypto-config/peerOrganizations/org1.example.com/ca/key.pem
        - name: FABRIC_CA_SERVER_DEBUG
          value: "true"
        - name: FABRIC_CA_SERVER_TLS_ENABLED
          value: "false"
        - name: FABRIC_CA_SERVER_TLS_CERTFILE
          value: /certs/ca0a-cert.pem
        - name: FABRIC_CA_SERVER_TLS_KEYFILE
          value: /certs/ca0a-key.pem
        - name: GODEBUG
          value: "netdns=go"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
```
*Note: The CA uses our shared filesystem.*  
*Note: The timezone configuration is important for certificate validation and expiration.*  

Now let's apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-ca_deploy.yaml
```

Create the file `kubernetes/blockchain-ca_svc.yaml` with the following `Service` description:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-ca
  labels:
    run: blockchain-ca
spec:
  type: ClusterIP
  selector:
    name: ca
  ports:
  - protocol: TCP
    port: 30054
    targetPort: 7054
    name: grpc
  - protocol: TCP
    port: 7054
    name: grpc1
```

Now, apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-ca_svc.yaml
```


### Step 7: Setting up Fabric Orderer

Create the file `kubernetes/blockchain-orderer_deploy.yaml` with the following `Deployment` description:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-orderer
spec:
  replicas: 3
  template:
    metadata:
      labels:
        name: orderer
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc

      containers:
      - name: orderer
        image: hyperledger/fabric-orderer:amd64-1.3.0
        command: ["sh", "-c", "orderer"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: ORDERER_CFG_PATH
          value: /fabric/
        - name: ORDERER_GENERAL_LEDGERTYPE
          value: file
        - name: ORDERER_FILELEDGER_LOCATION
          value: /fabric/ledger/orderer
        - name: ORDERER_GENERAL_BATCHTIMEOUT
          value: 1s
        - name: ORDERER_GENERAL_BATCHSIZE_MAXMESSAGECOUNT
          value: "10"
        - name: ORDERER_GENERAL_MAXWINDOWSIZE
          value: "1000"
        - name: CONFIGTX_GENERAL_ORDERERTYPE
          value: kafka
        - name: CONFIGTX_ORDERER_KAFKA_BROKERS
          value: "kafka1.local.parisi.biz:9092,kafka2.local.parisi.biz:9092,kafka3.local.parisi.biz:9092,kafka4.local.parisi.biz:9092"
        - name: ORDERER_KAFKA_RETRY_SHORTINTERVAL
          value: 1s
        - name: ORDERER_KAFKA_RETRY_SHORTTOTAL
          value: 30s
        - name: ORDERER_KAFKA_VERBOSE
          value: "true"
        - name: CONFIGTX_ORDERER_ADDRESSES
          value: "blockchain-orderer:31010"
        - name: ORDERER_GENERAL_LISTENADDRESS
          value: 0.0.0.0
        - name: ORDERER_GENERAL_LISTENPORT
          value: "31010"
        - name: ORDERER_GENERAL_LOGLEVEL
          value: debug
        - name: ORDERER_GENERAL_LOCALMSPDIR
          value: /fabric/crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/msp
        - name: ORDERER_GENERAL_LOCALMSPID
          value: OrdererMSP
        - name: ORDERER_GENERAL_GENESISMETHOD
          value: file
        - name: ORDERER_GENERAL_GENESISFILE
          value: /fabric/genesis.block
        - name: ORDERER_GENERAL_GENESISPROFILE
          value: initial
        - name: ORDERER_GENERAL_TLS_ENABLED
          value: "false"
        - name: GODEBUG
          value: "netdns=go"
        - name: ORDERER_GENERAL_LEDGERTYPE
          value: "ram"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
```
*Note: Because we're handling transactions, timezones needs to be in sync everywhere.*  
*Note: The Orderer also uses our shared filesystem.*  
*Note: Orderer is using Kafka.*  
*Note: Kafka Brokers previoulsy set on configtx are now listed under CONFIGTX_ORDERER_KAFKA_BROKERS environment variable.*  
*Note: We're using a deployment with 3 Orderers.*  

Let's apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-orderer_deploy.yaml
```

Create the file `kubernetes/blockchain-orderer_svc.yaml` with the following `Service` description:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-orderer
  labels:
    run: blockchain-orderer
spec:
  type: ClusterIP
  selector:
    name: orderer
  ports:
  - protocol: TCP
    port: 31010
    name: grpc
```
*Note: This service will Load Balance between the 3 Orderer Pods created in the previous Deployment.*  


Now, apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-orderer_svc.yaml
```

### Step 8: Org1MSP

- Create Org1MSP Peer1 Deployment  
Create the file `kubernetes/blockchain-org1peer1_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org1peer1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org1peer1
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ID
          value: blockchain-org1peer1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org1peer1:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org1peer1:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org1peer1:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org1peer1:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "true"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org1MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: info
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org1MSP Peer2 Deployment  
Create the file `kubernetes/blockchain-org1peer2_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org1peer2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org1peer2
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ID
          value: blockchain-org1peer2
        - name: CORE_PEER_ADDRESS
          value: blockchain-org1peer2:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org1peer2:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org1peer2:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org1peer2:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "true"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org1MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: info
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org1MSP Peer1 Service  
Create the file `kubernetes/blockchain-org1peer1_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org1peer1
  labels:
    run: blockchain-org1peer1
spec:
  type: ClusterIP 
  selector:
    name: org1peer1
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Create Org1MSP Peer2 Service  
Create the file `kubernetes/blockchain-org1peer2_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org1peer2
  labels:
    run: blockchain-org1peer2
spec:
  type: ClusterIP 
  selector:
    name: org1peer2
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Apply Configuration  
Now we're going to apply the previously created files:
```sh
kubectl apply -f kubernetes/blockchain-org1peer1_deploy.yaml
kubectl apply -f kubernetes/blockchain-org1peer2_deploy.yaml
kubectl apply -f kubernetes/blockchain-org1peer1_svc.yaml
kubectl apply -f kubernetes/blockchain-org1peer2_svc.yaml
```

### Step 9: Org2MSP

- Create Org2MSP Peer1 Deployment  
Create the file `kubernetes/blockchain-org2peer1_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org2peer1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org2peer1
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org2peer1
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org2peer1:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org2peer1:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org2peer1:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org2peer1:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org2MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org2MSP Peer2 Deployment  
Create the file `kubernetes/blockchain-org2peer2_deploy.yaml` the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org2peer2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org2peer2
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org2peer2
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org2peer2:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org2peer2:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org2peer2:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org2peer2:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org2MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org2MSP Peer1 Service  
Create the file `kubernetes/blockchain-org2peer1_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org2peer1
  labels:
    run: blockchain-org2peer1
spec:
  type: ClusterIP 
  selector:
    name: org2peer1
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Create Org2MSP Peer2 Service  
Create the file `kubernetes/blockchain-org2peer2_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org2peer2
  labels:
    run: blockchain-org2peer2
spec:
  type: ClusterIP 
  selector:
    name: org2peer2
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```

- Apply Configuration  
Now we're going to apply the previously created files:
```sh
kubectl apply -f kubernetes/blockchain-org2peer1_deploy.yaml
kubectl apply -f kubernetes/blockchain-org2peer2_deploy.yaml
kubectl apply -f kubernetes/blockchain-org2peer1_svc.yaml
kubectl apply -f kubernetes/blockchain-org2peer2_svc.yaml
```

### Step 10: Org3MSP

- Create Org3MSP Peer1 Deployment  
Create the file `kubernetes/blockchain-org3peer1_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org3peer1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org3peer1
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org3peer1
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org3peer1:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org3peer1:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org3peer1:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org3peer1:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org3MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org3MSP Peer2 Deployment  
Create the file `kubernetes/blockchain-org3peer2_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org3peer2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org3peer2
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org3peer2
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org3peer2:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org3peer2:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org3peer2:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org3peer2:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "true"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org3MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org3MSP Peer1 Service  
Create the file `kubernetes/blockchain-org3peer1_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org3peer1
  labels:
    run: blockchain-org3peer1
spec:
  type: ClusterIP 
  selector:
    name: org3peer1
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Create Org3MSP Peer2 Service  
Create the file `kubernetes/blockchain-org3peer2_svc.yaml` with `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org3peer2
  labels:
    run: blockchain-org3peer2
spec:
  type: ClusterIP 
  selector:
    name: org3peer2
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Apply Configuration  
Now we're going to apply the previously created files:
```sh
kubectl apply -f kubernetes/blockchain-org3peer1_deploy.yaml
kubectl apply -f kubernetes/blockchain-org3peer2_deploy.yaml
kubectl apply -f kubernetes/blockchain-org3peer1_svc.yaml
kubectl apply -f kubernetes/blockchain-org3peer2_svc.yaml
```

### Step 11: Org4MSP

- Create Org4MSP Peer1 Deployment  
Create the file `kubernetes/blockchain-org4peer1_deploy.yaml` the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org4peer1
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org4peer1
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org4peer1
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org4peer1:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org4peer1:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org4peer1:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org4peer1:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "false"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "true"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org4MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org4.example.com/peers/peer0.org4.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org4MSP Peer2 Deployment  
Create the file `kubernetes/blockchain-org4peer2_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-org4peer2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: org4peer2
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc
      - name: dockersocket
        hostPath:
          path: /var/run/docker.sock

      containers:
      - name: peer
        image: hyperledger/fabric-peer:amd64-1.3.0
        command: ["sh", "-c", "peer node start"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_ID
          value: blockchain-org4peer2
        - name: CORE_PEER_NETWORKID
          value: nid1
        - name: CORE_PEER_ADDRESS
          value: blockchain-org4peer2:30110
        - name: CORE_PEER_LISTENADDRESS
          value: 0.0.0.0:30110
        - name: CORE_PEER_EVENTS_ADDRESS
          value: 0.0.0.0:30111
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: blockchain-org4peer2:30110
        - name: CORE_PEER_GOSSIP_ENDPOINT
          value: blockchain-org4peer2:30110
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: blockchain-org4peer2:30110
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_SKIPHANDSHAKE
          value: "false"
        - name: CORE_PEER_COMMITTER_ENABLED
          value: "true"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_VM_ENDPOINT
          value: unix:///host/var/run/docker.sock
        - name: CORE_PEER_LOCALMSPID
          value: Org4MSP
        - name: CORE_PEER_MSPCONFIGPATH
          value: /fabric/crypto-config/peerOrganizations/org4.example.com/peers/peer1.org4.example.com/msp/
        - name: CORE_PEER_VALIDATOR_CONSENSUS_PLUGIN
          value: "pbft"
        - name: CORE_PBFT_GENERAL_MODE
          value: "classic"
        - name: CORE_PBFT_GENERAL_N
          value: "4"
        - name: CORE_LOGGING_LEVEL
          value: debug
        - name: CORE_LOGGING_PEER
          value: debug
        - name: CORE_LOGGING_CAUTHDSL
          value: debug
        - name: CORE_LOGGING_GOSSIP
          value: debug
        - name: CORE_LOGGING_LEDGER
          value: debug
        - name: CORE_LOGGING_MSP
          value: debug
        - name: CORE_LOGGING_POLICIES
          value: debug
        - name: CORE_LOGGING_GRPC
          value: debug
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: "hyperledgeruser"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: "hyperledgerpass"
        - name: FABRIC_CFG_PATH
          value: /etc/hyperledger/fabric/
        - name: ORDERER_URL
          value: blockchain-orderer:31010
        - name: GODEBUG
          value: "netdns=go"
        - name: CORE_VM_DOCKER_ATTACHSTDOUT
          value: "true"
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
        - mountPath: /host/var/run/docker.sock
          name: dockersocket
      - name: couchdb
        image: hyperledger/fabric-couchdb:amd64-0.4.14
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: COUCHDB_USER
          value: "hyperledgeruser"
        - name: COUCHDB_PASSWORD
          value: "hyperledgerpass"
```
*Note: The peer uses our shared filesystem.*  
*Note: Because we're handling with transactions, its important that every pod is running in the same timezone. Pay attention to the TZ environment variable.*  
*Note: CORE_PEER_GOSSIP_BOOTSTRAP, CORE_PEER_GOSSIP_ENDPOINT and CORE_PEER_GOSSIP_EXTERNALENDPOINT are critical for the Hyperledger Fabric Service Discovery to work.*  
*Note: Volume dockersocket is used in order for the peer to have access to the docker daemon running on the host the peer is running, to be able to launch the chaincode container*  
*Note: The chaincode container will be launched directly into Docker Daemon, and will not show up in Kubernetes.*  
*Note: There is a sidecar container running CouchDB. There are environment variables setting the peer to use this CouchDB instance.*  

- Create Org4MSP Peer1 Service  
Create the file `kubernetes/blockchain-org4peer1_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org4peer1
  labels:
    run: blockchain-org4peer1
spec:
  type: ClusterIP 
  selector:
    name: org4peer1
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Create Org4MSP Peer2 Service  
Create the file `kubernetes/blockchain-org4peer2_svc.yaml` with the `Service` below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-org4peer2
  labels:
    run: blockchain-org4peer2
spec:
  type: ClusterIP 
  selector:
    name: org4peer2
  ports:
  - protocol: TCP
    port: 30110
    name: grpc
  - protocol: TCP
    port: 30111
    name: events
  - protocol: TCP
    port: 5984
    name: couchdb
```
- Apply Configuration  
Now we're going to apply the previously created files:
```sh
kubectl apply -f kubernetes/blockchain-org4peer1_deploy.yaml
kubectl apply -f kubernetes/blockchain-org4peer2_deploy.yaml
kubectl apply -f kubernetes/blockchain-org4peer1_svc.yaml
kubectl apply -f kubernetes/blockchain-org4peer2_svc.yaml
```

### Step 12: Create Channel
Now its time to create our channel:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
cd /fabric
configtxgen -profile FourOrgsChannel -outputCreateChannelTx ${CHANNEL_NAME}.tx -channelID ${CHANNEL_NAME}

export ORDERER_URL="blockchain-orderer:31010"
export CORE_PEER_ADDRESSAUTODETECT="false"
export CORE_PEER_NETWORKID="nid1"
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
peer channel create -o ${ORDERER_URL} -c ${CHANNEL_NAME} -f /fabric/${CHANNEL_NAME}.tx 
exit
```

### Step 13: Join Channel

- Org1MSP  
Lets join Org1MSP to our channel:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
export CORE_PEER_NETWORKID="nid1"
export ORDERER_URL="blockchain-orderer:31010"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_MSPID="Org1MSP"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp"
export CORE_PEER_ADDRESS="blockchain-org1peer1:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
export CORE_PEER_ADDRESS="blockchain-org1peer2:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
exit
```
- Org2MSP  
Lets join Org2MSP to our channel:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
export CORE_PEER_NETWORKID="nid1"
export ORDERER_URL="blockchain-orderer:31010"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_MSPID="Org2MSP"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp"
export CORE_PEER_ADDRESS="blockchain-org2peer1:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
export CORE_PEER_ADDRESS="blockchain-org2peer2:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
exit
```
- Org3MSP  
Lets join Org3MSP to our channel:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
export CORE_PEER_NETWORKID="nid1"
export ORDERER_URL="blockchain-orderer:31010"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_LOCALMSPID="Org3MSP"
export CORE_PEER_MSPID="Org3MSP"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp"
export CORE_PEER_ADDRESS="blockchain-org3peer1:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
export CORE_PEER_ADDRESS="blockchain-org3peer2:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
exit
```
- Org4MSP  
Lets join Org4MSP to our channel:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
export CORE_PEER_NETWORKID="nid1"
export ORDERER_URL="blockchain-orderer:31010"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_LOCALMSPID="Org4MSP"
export CORE_PEER_MSPID="Org4MSP"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org4.example.com/users/Admin@org4.example.com/msp"
export CORE_PEER_ADDRESS="blockchain-org4peer1:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
export CORE_PEER_ADDRESS="blockchain-org4peer2:30110"

peer channel fetch newest -o ${ORDERER_URL} -c ${CHANNEL_NAME}
peer channel join -b ${CHANNEL_NAME}_newest.block

rm -rf /${CHANNEL_NAME}_newest.block
exit
```

### Step 14: Install Chaincode

- Org1MSP  
Lets install our chaincode on Org1MSP Peers:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp -r /fabric/config/chaincode $GOPATH/src/
export CHAINCODE_NAME="cc"
export CHAINCODE_VERSION="1.0"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp"
export CORE_PEER_LOCALMSPID="Org1MSP"

export CORE_PEER_ADDRESS="blockchain-org1peer1:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

export CORE_PEER_ADDRESS="blockchain-org1peer2:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

exit
```
- Org2MSP  
Lets install our chaincode on Org2MSP Peers:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp -r /fabric/config/chaincode $GOPATH/src/
export CHAINCODE_NAME="cc"
export CHAINCODE_VERSION="1.0"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp"
export CORE_PEER_LOCALMSPID="Org2MSP"

export CORE_PEER_ADDRESS="blockchain-org2peer1:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

export CORE_PEER_ADDRESS="blockchain-org2peer2:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

exit
```
- Org3MSP  
Lets install our chaincode on Org3MSP Peers:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp -r /fabric/config/chaincode $GOPATH/src/
export CHAINCODE_NAME="cc"
export CHAINCODE_VERSION="1.0"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp"
export CORE_PEER_LOCALMSPID="Org3MSP"

export CORE_PEER_ADDRESS="blockchain-org3peer1:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

export CORE_PEER_ADDRESS="blockchain-org3peer2:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

exit
```
- Org4MSP  
Lets install our chaincode on Org4MSP Peers:
```sh
kubectl exec -it fabric-tools -- /bin/bash
cp -r /fabric/config/chaincode $GOPATH/src/
export CHAINCODE_NAME="cc"
export CHAINCODE_VERSION="1.0"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org4.example.com/users/Admin@org4.example.com/msp"
export CORE_PEER_LOCALMSPID="Org4MSP"

export CORE_PEER_ADDRESS="blockchain-org4peer1:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

export CORE_PEER_ADDRESS="blockchain-org4peer2:30110"
peer chaincode install -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -p chaincode_example02/

exit
```

### Step 15: Instantiate Chaincode

Now its time to instantiate our chaincode:
```sh
kubectl exec -it fabric-tools -- /bin/bash
export CHANNEL_NAME="channel1"
export CHAINCODE_NAME="cc"
export CHAINCODE_VERSION="1.0"
export FABRIC_CFG_PATH="/etc/hyperledger/fabric"
export CORE_PEER_MSPCONFIGPATH="/fabric/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp"
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_ADDRESS="blockchain-org1peer1:30110"
export ORDERER_URL="blockchain-orderer:31010"

peer chaincode instantiate -o ${ORDERER_URL} -C ${CHANNEL_NAME} -n ${CHAINCODE_NAME} -v ${CHAINCODE_VERSION} -P "AND('Org1MSP.member','Org2MSP.member','Org3MSP.member','Org4MSP.member')" -c '{"Args":["init","a","300","b","600"]}'
exit
```
*Note: The policy -P is set using AND. This will set the policy in a way that at least 1 peer from each Org will need to endorse the transaction.*  
*Note: Because of this policy, every transaction sent to the network will have to be sent to at least 1 peer from each organization.*  
*Note: As we're using Balance Transfer example, we're starting A with 300 and B with 600.*  

### Step 16: AnchorPeers

Now we need to update our channel configuration to reflect our Anchor Peers:
```sh
pod=$(kubectl get pods | grep blockchain-org1peer1 | awk '{print $1}')
kubectl exec -it $pod -- peer channel update -f /fabric/Org1MSPanchors.tx -c channel1 -o blockchain-orderer:31010 

pod=$(kubectl get pods | grep blockchain-org2peer1 | awk '{print $1}')
kubectl exec -it $pod -- peer channel update -f /fabric/Org2MSPanchors.tx -c channel1 -o blockchain-orderer:31010

pod=$(kubectl get pods | grep blockchain-org3peer1 | awk '{print $1}')
kubectl exec -it $pod -- peer channel update -f /fabric/Org3MSPanchors.tx -c channel1 -o blockchain-orderer:31010

pod=$(kubectl get pods | grep blockchain-org4peer1 | awk '{print $1}')
kubectl exec -it $pod -- peer channel update -f /fabric/Org4MSPanchors.tx -c channel1 -o blockchain-orderer:31010 
```
*Note: This step is very important for the Hyperledger Fabric Service Discovery to work properly.*

### Step 17: Deploy Hyperledger Explorer

Fabric Explorer needs a PostgreSQL Database as its backend. In order to deploy, we'll create the file `kubernetes/blockchain-explorer-db_deploy.yaml` with the following `Deployment`:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-explorer-db
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: explorer-db
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc

      containers:
      - name: postgres
        image: postgres:10.4-alpine
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: DATABASE_DATABASE
          value: fabricexplorer
        - name: DATABASE_USERNAME
          value: hppoc
        - name: DATABASE_PASSWORD
          value: password
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
```
*Note: The timezone is also important here.*  
*Note: This pod will need Internet access.*  

Now we're going to apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-explorer-db_deploy.yaml
```

After that, we need to create the `Service` entry for our database. To do that lets create the file `kubernetes/blockchain-explorer-db_svc.yaml` as below:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: blockchain-explorer-db
  labels:
    run: explorer-db
spec:
  type: ClusterIP 
  selector:
    name: explorer-db
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432 
    name: pgsql
```

Now we're going to apply the configuration:
```sh
kubectl apply -f kubernetes/blockchain-explorer-db_svc.yaml
```

Now, before proceeding, make sure the PostgreSQL Pod is running. We need to create the tables and artifacts for Hyperledger Explorer in our database:
```sh
pod=$(kubectl get pods | grep blockchain-explorer-db | awk '{print $1}')
kubectl exec -it $pod -- /bin/bash
mkdir -p /fabric/config/explorer/db/
mkdir -p /fabric/config/explorer/app/
cd /fabric/config/explorer/db/
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/createdb.sh
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/explorerpg.sql
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/processenv.js
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/updatepg.sql
apk update
apk add jq
apk add nodejs
apk add sudo
rm -rf /var/cache/apk/*
chmod +x ./createdb.sh
./createdb.sh
exit
```
Now, we're going to create the config file with our Hyperledger Network description to use on Hyperledger Explorer. In order to do that, we'll create the file `config/explorer/app/config.json` with the following configuration:
```json
{
  "network-configs": {
    "network-1": {
      "version": "1.0",
      "clients": {
        "client-1": {
          "tlsEnable": false,
          "organization": "Org1MSP",
          "channel": "channel1",
          "credentialStore": {
            "path": "./tmp/credentialStore_Org1/credential",
            "cryptoStore": {
              "path": "./tmp/credentialStore_Org1/crypto"
            }
          }
        }
      },
      "channels": {
        "channel1": {
          "peers": {
            "blockchain-org1peer1": {},
            "blockchain-org2peer1": {},
            "blockchain-org3peer1": {},
            "blockchain-org4peer1": {},
            "blockchain-org1peer2": {},
            "blockchain-org2peer2": {},
            "blockchain-org3peer2": {},
            "blockchain-org4peer2": {}
          },
          "orderers": {
            "blockchain-orderer" : {}
          },
          "connection": {
            "timeout": {
              "peer": {
                "endorser": "6000",
                "eventHub": "6000",
                "eventReg": "6000"
              }
            }
          }
        }
      },
      "organizations": {
        "Org1MSP": {
          "mspid": "Org1MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/signcerts"
          }
        },
        "Org2MSP": {
          "mspid": "Org2MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp/signcerts"
          }
        },
        "Org3MSP": {
          "mspid": "Org3MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp/signcerts"
          }
        },
        "Org4MSP": {
          "mspid": "Org4MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org4.example.com/users/Admin@org4.example.com/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org4.example.com/users/Admin@org4.example.com/msp/signcerts"
          }
        },
        "OrdererMSP": {
          "mspid": "OrdererMSP",
          "adminPrivateKey": {
            "path":
              "/fabric/crypto-config/ordererOrganizations/example.com/users/Admin@example.com/msp/keystore"
          }
        }
      },
      "peers": {
        "blockchain-org1peer1": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org1peer1:30110",
          "eventUrl": "grpc://blockchain-org1peer1:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org1.example.com"
          }
        },
        "blockchain-org2peer1": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org2peer1:30110",
          "eventUrl": "grpc://blockchain-org2peer1:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org2.example.com"
          }
        },
        "blockchain-org3peer1": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org3peer1:30110",
          "eventUrl": "grpc://blockchain-org3peer1:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org3.example.com"
          }
        },
        "blockchain-org4peer1": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org4.example.com/peers/peer0.org4.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org4peer1:30110",
          "eventUrl": "grpc://blockchain-org4peer1:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org4.example.com"
          }
        },
        "blockchain-org1peer2": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org1.example.com/peers/peer1.org1.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org1peer2:30110",
          "eventUrl": "grpc://blockchain-org1peer2:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org1.example.com"
          }
        },
        "blockchain-org2peer2": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org2peer2:30110",
          "eventUrl": "grpc://blockchain-org2peer2:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org2.example.com"
          }
        },
        "blockchain-org3peer2": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org3.example.com/peers/peer1.org3.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org3peer2:30110",
          "eventUrl": "grpc://blockchain-org3peer2:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org3.example.com"
          }
        },
        "blockchain-org4peer2": {
          "tlsCACerts": {
            "path":
              "/fabric/crypto-config/peerOrganizations/org4.example.com/peers/peer1.org4.example.com/tls/ca.crt"
          },
          "url": "grpc://blockchain-org4peer2:30110",
          "eventUrl": "grpc://blockchain-org4peer2:30111",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org4.example.com"
          }
        }
      },
      "orderers": {
        "blockchain-orderer": {
          "url": "grpc://blockchain-orderer:31010"
        }
      }
    }
  },
  "configtxgenToolPath": "/fabric-path/workspace/fabric-samples/bin",
  "license": "Apache-2.0"
}
```

After creating the file, its time to copy it to our shared filesystem:
```sh
kubectl cp config/explorer/app/config.json fabric-tools:/fabric/config/explorer/app/
```  

Create the `config/explorer/app/run.sh` as below:
```sh
#!/bin/sh
mkdir -p /opt/explorer/app/platform/fabric/
mkdir -p /tmp/

mv /opt/explorer/app/platform/fabric/config.json /opt/explorer/app/platform/fabric/config.json.vanilla
cp /fabric/config/explorer/app/config.json /opt/explorer/app/platform/fabric/config.json

cd /opt/explorer
node $EXPLORER_APP_PATH/main.js && tail -f /dev/null
```

After creating the file, its time to copy it to our shared filesystem:
```sh
chmod +x config/explorer/app/run.sh
kubectl cp config/explorer/app/run.sh fabric-tools:/fabric/config/explorer/app/
```

Now its time to create our Hyperledger Explorer application `Deployment` by creating the file `kubernetes/blockchain-explorer-app_deploy.yaml` as below:
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: blockchain-explorer-app
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: explorer
    spec:
      volumes:
      - name: fabricfiles
        persistentVolumeClaim:
          claimName: fabric-pvc

      containers:
      - name: explorer
        image: hyperledger/explorer:latest
        command: ["sh" , "-c" , "/fabric/config/explorer/app/run.sh"]
        env:
        - name: TZ
          value: "America/Sao_Paulo"
        - name: DATABASE_HOST
          value: blockchain-explorer-db
        - name: DATABASE_USERNAME
          value: hppoc
        - name: DATABASE_PASSWORD
          value: password
        volumeMounts:
        - mountPath: /fabric
          name: fabricfiles
```
*Note: Again setting up the timezone as the reports might get impacted.*  
*Note: This deployment will have access to the share filesystem as the startup script and config files are store there.*  

Now its time to apply our deploy:
```sh
kubectl apply -f kubernetes/blockchain-explorer-app_deploy.yaml
```

## CLEANUP

Now, to leave our environment clean, we're going to remove our helper `Pod`:
```sh
kubectl delete -f kubernetes/fabric-tools.yaml
```

## VALIDATING

Now, we're going to run 2 transactions. The first one we'll move 50 from `A` to `B`. The second one we'll move 33 from `B` to `A`: 
```sh
pod=$(kubectl get pods | grep blockchain-org1peer1 | awk '{print $1}')
kubectl exec -it $pod -- /bin/bash
peer chaincode invoke --peerAddresses blockchain-org1peer1:30110 --peerAddresses blockchain-org2peer1:30110 --peerAddresses blockchain-org3peer1:30110 --peerAddresses blockchain-org4peer1:30110 -o blockchain-orderer:31010 -C channel1 -n cc -c '{"Args":["invoke","a","b","50"]}'

peer chaincode invoke --peerAddresses blockchain-org1peer1:30110 --peerAddresses blockchain-org2peer1:30110 --peerAddresses blockchain-org3peer1:30110 --peerAddresses blockchain-org4peer1:30110 -o blockchain-orderer:31010 -C channel1 -n cc -c '{"Args":["invoke","b","a","33"]}'

exit
```
*Note: The invoke command is using --peerAddresses parameter four times, in order to send the transaction to at least one peer from each organization.*  
*Note: The first transaction might take a little bit to go through.*  

Now we're going to check our balance. As stated before, we've started `A` with 300 and `B` with 600: 
```sh
pod=$(kubectl get pods | grep blockchain-org1peer1 | awk '{print $1}')
kubectl exec -it $pod -- /bin/bash

peer chaincode query -C channel1 -n cc -c '{"Args":["query","a"]}'

peer chaincode query -C channel1 -n cc -c '{"Args":["query","b"]}'

exit
```
*Note: A should return 283 and B should return 617.*  

We can also check the network status as well as the transactions on Hyperledger Explorer:
```sh
pod=$(kubectl get pods | grep blockchain-explorer-app | awk '{print $1}')
kubectl port-forward $pod 8080:8080
```

Now open your browser to [http://127.0.0.1:8080/](http://127.0.0.1:8080/).
In the first window you can see your network status and transactions as below:  

![slide6.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide6.jpg)

You can also click on transactions tab, and check for a transaction as below:  

![slide7.jpg](https://github.com/feitnomore/hyperledger-fabric-kubernetes/raw/master/images/slide7.jpg)

*Note: You can see here that the transactions got endorsed by all the 4 Organizations.*  

## Reference Links

* [Hyperledger Fabric](https://hyperledger-fabric.readthedocs.io/en/release-1.3/)
* [Hyperledger Explorer](https://www.hyperledger.org/projects/explorer)
* [Apache CouchDB](http://couchdb.apache.org/)
* [Apache Kafka](https://kafka.apache.org/)
* [Kubernetes Concepts](https://kubernetes.io/docs/concepts/)
