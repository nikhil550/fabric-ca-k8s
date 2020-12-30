# **Hyperledger Fabric on IBM CLoud

## Connect to your IBM Kubernetes service

When you have connected, you will be able to run the following comand
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
The expected output from the above command should be similar to:

In the terminal, execute the following command to set up a storage volume that is connected to the PVC we created in the previous step.  This is one method of persisting data in the cloud even if your Kubernetes pods restart.  ***Important Note: Data will NOT persist if you delete the persistent volume claim!  If you plan to start over and you run `minikube delete` your persistent volume and persistent volume claim will be deleted!***

```
kubectl apply -f redis-storage.yaml
```
Expected output after running the above command is similar to below.


In the terminal, run the following command to check the status of the pod.
```
kubectl get pod
```
Expected output after running the above command is similar to below.



## Deploy the CA's

We will deploy two CA's, one for the peer and one for the orderer. Copy the `fabric-ca-server-config.yaml` file for the two  into the redis container:

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

You can interact with the CA by copying the TLS certificates of the two CA's from the redis database to your local filesystem.
```
kubectl cp redis:/data/redis/hyperledger/orderer-ca/tls-cert.pem $PWD/hyperledger/orderer-ca/tls-cert.pem
```
and
```
kubectl cp redis:/data/redis/hyperledger/peer-ca/tls-cert.pem $PWD/hyperledger/peer-ca/tls-cert.pem
```

### enroll the CA admins

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

## Register the peer identities

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


## Register the orderer identities

We can follow the same steps to set up the ordering node using the orderer ca.
```
export FABRIC_CA_CLIENT_HOME=$PWD/symbridge-orderer-ca/admin
```

We will register the peer admin first. This identity will be used to operate the peer:
```
fabric-ca-client register --caname orderer-ca --id.name ordereradmin --id.secret ordererpw --id.type admin --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

We can now register the peer identity:
```
fabric-ca-client register --caname orderer-ca --id.name orderer --id.secret ordererpw --id.type orderer --tls.certfiles ${PWD}/hyperledger/orderer-ca/tls-cert.pem
```

## Enroll the peer identities

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


## Enroll the orderer identities

Now that we have registered the identity, we can generate the certificates for our nodes. First, enroll the peer admin:

```
fabric-ca-client enroll -u https://peeradmin:peeradminpw@159.122.186.71:30122 --caname peer-ca -M $PWD/symbridge-peer-ca/peeradmin/msp --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

We can now enroll the peer. We will complete two enrollments, once against the enrollment CA to generate the peer signing certificate, and again to generate the peer TLS certificates.
```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30122 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/msp --csr.hosts peer.symbridge.com --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```

```
fabric-ca-client enroll -u https://peer:peerpw@159.122.186.71:30122 --caname peer-ca -M ${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls --enrollment.profile tls --csr.hosts 159.122.186.71 --csr.hosts 127.0.0.1 --tls.certfiles ${PWD}/hyperledger/peer-ca/tls-cert.pem
```


## Deploy the peer

We now have created the crypto material required to deploy the peer. Copy those certificates and keys from the local filesystem into kubernetes.
```
kubectl cp $PWD/hyperledger/peer/ redis:/data/redis/hyperledger/
```
```
kubectl apply -f fabric-peer-deployment.yaml
```


## Interact with the peer

If the peer is deployed, you can use set environment variables to interact with the peer

```
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="SymbridgePeerMSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/signcerts/cert.pem
export CORE_PEER_MSPCONFIGPATH=${PWD}/symbridge-peer-ca/peeradmin/msp
export CORE_PEER_ADDRESS=159.122.186.71:30202
```
