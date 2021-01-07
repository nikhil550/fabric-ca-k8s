# Hyperledger Fabric on IBM Cloud

## Connect to your IBM Kubernetes service

When you have connected, you will be able to run the following command
```
kubectl get all
```
Expected output is similar to below, note your ClusterIP will be different.


## Set up the persistent volume claim and provision storage

Now run the following kubectl command in the terminal to provision a persistent volume claim (PVC).
```
kubectl apply -f setup-pvc.yaml
```
Now run the following command in the terminal to confirm your persistent volume claim has been set up.
```
kubectl get pvc
```

Run the following command to deploy Redis storage.
```
kubectl apply -f redis-storage.yaml
```

In the terminal, run the following command to check the status of the pod.
```
kubectl get pod
```
Expected output after running the above command is similar to below.


## Deploy the CA's

We will deploy two CA's, one for the peer and one for the orderer. Copy the `fabric-ca-server-config.yaml` file for the two  into the Redis container:

```
kubectl cp $PWD/hyperledger/ redis:/data/redis/
```


Now we are ready to start the two CA's:
```
kubectl apply -f peer-ca-deployment.yaml
```
and
```
kubectl apply -f orderer-ca-deployment.yaml
```

You can use the following command to make sure that the two pods are running
```
kubectl get pods
```

You can interact with the CA by copying the TLS certificates of the two CA's from the Redis database to your local filesystem.
```
kubectl cp redis:/data/redis/hyperledger/orderer-ca/tls-cert.pem $PWD/hyperledger/orderer-ca/tls-cert.pem
```
and
```
kubectl cp redis:/data/redis/hyperledger/peer-ca/tls-cert.pem $PWD/hyperledger/peer-ca/tls-cert.pem
```

### Enroll the CA admins

Ensure that the Fabric binaries and configuration files are copied to your local folder to ``$PWD/bin`` and `$PWD/config`. You can then use the Fabric CA by setting the following
```
export PATH=${PWD}/bin:${PWD}:$PATH
export FABRIC_CFG_PATH=$PWD/config/
```

You can new enroll the peer ca admin:
```
mkdir -p $PWD/symbridge-peer-ca/admin
export FABRIC_CA_CLIENT_HOME=$PWD/symbridge-peer-ca/admin
fabric-ca-client enroll -d -u https://admin:adminpw@159.122.186.71:30102 --caname peer-ca --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

and the orderer-ca admin.
```
mkdir -p $PWD/symbridge-orderer-ca/admin
export FABRIC_CA_CLIENT_HOME=$PWD/symbridge-orderer-ca/admin
fabric-ca-client enroll -d -u https://admin:adminpw@159.122.186.71:30122 --caname orderer-ca --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```


## Using the peer CA

### Register node and user identities

The CA admin lets us register the peer identities, and peer admin. Set the Fabric CA home path to use the CA admin of the peer CA:
```
export FABRIC_CA_CLIENT_HOME=$PWD/symbridge-peer-ca/admin
```

We will register the peer admin first. This identity will be used to operate the peer:
```
fabric-ca-client register --caname peer-ca --id.name peeradmin --id.secret peeradminpw --id.type admin --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

We can now register the peer identity:
```
fabric-ca-client register --caname peer-ca --id.name peer --id.secret peerpw --id.type peer --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```


### Enroll the peer identities

Now that we have registered the identity, we can generate the certificates for our nodes. First, enroll the peer admin:

```
fabric-ca-client enroll -u https://peeradmin:peeradminpw@159.122.186.71:30102 --caname peer-ca -M $PWD/symbridge-peer-ca/peeradmin/msp --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

We can now enroll the peer. We will complete two enrollments, once against the enrollment CA to generate the peer signing certificate, and again to generate the peer TLS certificates.
```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30102 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp --csr.hosts peer.symbridge.com --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30102 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls --enrollment.profile tls --csr.hosts 159.122.186.71 --csr.hosts 127.0.0.1 --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

### Rename some files

Rename your TLS private key `server.key` to make it easier to work with.

### Node OU configuration file

To configure the Node OU feature, we need to create and copy the following file into the MSP file of each identity.

Run the following command to create the file:
```
echo 'NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: cacerts/159-122-186-71-30102-peer-ca.pem
    OrganizationalUnitIdentifier: client
  PeerOUIdentifier:
    Certificate: cacerts/159-122-186-71-30102-peer-ca.pem
    OrganizationalUnitIdentifier: peer
  AdminOUIdentifier:
    Certificate: cacerts/159-122-186-71-30102-peer-ca.pem
    OrganizationalUnitIdentifier: admin
  OrdererOUIdentifier:
    Certificate: cacerts/159-122-186-71-30102-peer-ca.pem
    OrganizationalUnitIdentifier: orderer' > ${PWD}/hyperledger/peer/config.yaml
```

You can then run the following commands to copy the configuration file into the correct msp folders:

```
cp ${PWD}/hyperledger/peer/config.yaml ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp/config.yaml
cp ${PWD}/hyperledger/peer/config.yaml $PWD/symbridge-peer-ca/peeradmin/msp/config.yaml
```

## Using the orderer CA

### Register the node and user identities

We can follow the same steps to set up the ordering node using the orderer ca.
```
export FABRIC_CA_CLIENT_HOME=$PWD/symbridge-orderer-ca/admin
```

We will register the peer admin first. This identity will be used to operate the peer:
```
fabric-ca-client register --caname orderer-ca --id.name osadmin --id.secret osadminpw --id.type admin --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

We can now register the peer identity:
```
fabric-ca-client register --caname orderer-ca --id.name orderer --id.secret ordererpw --id.type orderer --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```


### Enroll the orderer identities

Now that we have registered the identity, we can generate the certificates for our nodes. First, enroll the peer admin:

```
fabric-ca-client enroll -u https://osadmin:osadminpw@159.122.186.71:30122 --caname orderer-ca -M $PWD/orderer-peer-ca/ordereradmin/msp --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

We can now enroll the peer. We will complete two enrollments, once against the enrollment CA to generate the peer signing certificate, and again to generate the peer TLS certificates.
```
fabric-ca-client enroll -u https://orderer:ordererpw@159.122.186.71:30122 --caname orderer-ca -M ${PWD}/hyperledger/orderer/ordererOrganizations/example.com/orderers/orderer.example.com/msp --csr.hosts orderer.symbridge.com --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

```
fabric-ca-client enroll -u https://orderer:ordererpw@159.122.186.71:30122 --caname orderer-ca -M ${PWD}/hyperledger/orderer/ordererOrganizations/example.com/orderers/orderer.example.com/tls --enrollment.profile tls --csr.hosts 159.122.186.71 --csr.hosts 127.0.0.1 --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

### Rename some files

Rename your TLS private key `server.key` to make it easier to work with.

### Node OU configuration file

To configure the Node OU feature, we need to create and copy the following file into the MSP file of each identity.

Run the following command to create the file:
```
echo 'NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: cacerts/159-122-186-71-30122-orderer-ca.pem
    OrganizationalUnitIdentifier: client
  PeerOUIdentifier:
    Certificate: cacerts/159-122-186-71-30122-orderer-ca.pem
    OrganizationalUnitIdentifier: peer
  AdminOUIdentifier:
    Certificate: cacerts/159-122-186-71-30122-orderer-ca.pem
    OrganizationalUnitIdentifier: admin
  OrdererOUIdentifier:
    Certificate: cacerts/159-122-186-71-30122-orderer-ca.pem
    OrganizationalUnitIdentifier: orderer' > ${PWD}/hyperledger/orderer/config.yaml
```

You can then run the following commands to copy the configuration file into the correct MSP folders:

```
cp ${PWD}/hyperledger/orderer/config.yaml ${PWD}/hyperledger/orderer/ordererOrganizations/example.com/orderers/orderer.example.com/msp/config.yaml
cp ${PWD}/hyperledger/orderer/config.yaml $PWD/symbridge-orderer-ca/ordereradmin/msp/config.yaml
```


## Create the orderer system channel genesis block

In order to deploy an ordering node, we will need to create a genesis block. Set the fabric config path to the configuration file:
```
export FABRIC_CFG_PATH=${PWD}/configtx
```

```
configtxgen -profile TwoOrgsOrdererGenesis -channelID system-channel -outputBlock ./hyperledger/orderer/genesis.block
```

## Deploy the orderer

We now have the crypto material and the genesis block required to deploy the ordering node. Copy the certifcates, keys, and block into Redis database.

```
kubectl cp $PWD/hyperledger/orderer/ redis:/data/redis/hyperledger/
```
```
kubectl apply -f fabric-orderer-deployment.yaml
```

## Deploy the peer

We now have created the crypto material required to deploy the peer. Copy those certificates and keys from the local filesystem into the Redis databaase.
```
kubectl cp $PWD/hyperledger/peer/ redis:/data/redis/hyperledger/
```
```
kubectl apply -f fabric-peer-deployment.yaml
```


## Create a channel

Make sure the fabric config path is set to the confictx.yaml file.
```
export FABRIC_CFG_PATH=${PWD}/configtx
```

Use the file to create the channel creation artifact:
```
configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel1.tx -channelID channel1
```

set the environment variables to talk to our peer:
```
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="SymbridgePeerMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/signcerts/cert.pem
export CORE_PEER_MSPCONFIGPATH=${PWD}/symbridge-peer-ca/peeradmin/msp
export CORE_PEER_ADDRESS=159.122.186.71:30202
```
Also reset the Fabric config path:
```
export FABRIC_CFG_PATH=${PWD}/config
```

Now create the channel, pointing to the ordering node that we created. The `--cafile` points to the TLS certificate of the ordering node.
```
peer channel create -o 159.122.186.71:30222  -c channel1 -f ./channel1.tx --outputBlock ./channel1.block --tls --cafile ${PWD}/hyperledger/orderer/ordererOrganizations/example.com/orderers/orderer.example.com/tls/tlscacerts/tls-159-122-186-71-30122-orderer-ca.pem
```

This command will return a block from the orderer. Run the following command to join the peer to the channel:
```
peer channel join -b ./channel1.block
```

You can now verify that the channel was joined:
```
peer channel list
```
The response will be similar to the following:
```
2020-12-30 17:12:08.666 EST [channelCmd] InitCmdFactory -> INFO 001 Endorser and orderer connections initialized
Channels peers has joined:
channel1
```

## Deploy a smart contract

Package the smart contract in the fabric samples directory:
```
peer lifecycle chaincode package basic.tar.gz --path ../fabric-samples/asset-transfer-basic/chaincode-go/ --lang golang --label basic_1.0
```

```
peer lifecycle chaincode install basic.tar.gz
```

## Deploy a peer using TLS secrets

Use the peer identity to enroll a new peer identity.

```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30102 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp --csr.hosts peer.symbridge.com --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30102 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls --enrollment.profile tls --csr.hosts 159.122.186.71 --csr.hosts 127.0.0.1 --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

Rename the keys cacert.pem, key.pem, and cert.pem to make them easier to work with.

For the cacert:
```
kubectl create secret generic peer2-cacert --from-file=cacert.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp/cacerts/cacert.pem
```
signcert:
```
kubectl create secret generic peer2-cert --from-file=cert.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp/signcerts/cert.pem
```

and private key:
```
kubectl create secret generic peer2-key --from-file=key.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/msp/keystore/key.pem
```

Run the following command to create the configfile file:
```
echo 'NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: cacerts/cacert.pem
    OrganizationalUnitIdentifier: client
  PeerOUIdentifier:
    Certificate: cacerts/cacert.pem
    OrganizationalUnitIdentifier: peer
  AdminOUIdentifier:
    Certificate: cacerts/cacert.pem
    OrganizationalUnitIdentifier: admin
  OrdererOUIdentifier:
    Certificate: cacerts/cacert.pem
    OrganizationalUnitIdentifier: orderer' > ${PWD}/hyperledger/peer2/config.yaml
```

```
kubectl create secret generic peer2-configfile --from-file=config.yaml=./config.yaml
```

We need to do the same process with the TLS certs:

For the TLS cacert:
```
kubectl create secret generic peer2-tlscacert --from-file=tlscacert.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/tlscacerts/tlscacert.pem
```

TLS signcert:
```
kubectl create secret generic peer2-tlscert --from-file=cert.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/signcerts/cert.pem
```

and TLS private key:
```
kubectl create secret generic peer2-tlskey --from-file=key.pem=./hyperledger/peer/peerOrganizations/org1.example.com/peers/peer2.org1.example.com/tls/keystore/key.pem
```
