# **Hyperledger Fabric Certificate Authority on Kubernetes (k8s)**

Deploy a Hyperledger Fabric Certificate Authority using Native Kubernetes (k8s) for Production Hyperledger Fabric Networks Deployed on Cloud Infrastructure.
# ***IMPORTANT NOTE: DO NOT USE THIS CONFIGURATION IN PRODUCTION AS TLS IS NOT ENABLED, FOR PRACTICE ONLY!***

If you do not wish to use minikube and would rather deploy in the cloud, you need to ensure that you can provision storage and a peristent volume claim against that storage as each cloud provider is slightly different.  You must be able to access your cluster before proceeding if using cloud providers such as IBM, Google Cloud, AWS, Digital Ocean, or Azure.  You can check access by running `kubectl get all` in your terminal after you have downloaded the .kubeconfig file from your provider.

## **Purpose**
Learn how to configure and deploy a Hyperledger Fabric Certificate Authority on Kubernetes for a production network and manage cryptographic identities without using cryptogen.

## **Key Concepts**
Note: This tutorial assumes a novice level of experience with Kubernetes and kubectl.  We are going to make it easy enough that if you do not have this experience, you can learn it on the fly.  Please refer to the Kubernetes links throughout this tutorial for additional information if you get stuck, see errors, or have difficulties.

**Kubernetes** (a.k.a. k8s) [Kubernetes Documentation](https://kubernetes.io/)
- minikube (for deployment of a k8s cluster on your local machine for testing and this tutorial)
- kubectl cli commands and help
- kubernetes persistent volumes
- kubernetes persistent volume claims
- deploying storage

**Docker**

**Hyperledger Fabric Certificate Authority**
 - Fabric CA Server
 - Fabric CA Client
 - Membership Service Provider (MSP)

## Step-by-Step Tutorial
### At the time of this update, this tutorial was developed using:
- Mac OS Catalina 10.15.4
- minikube version: v1.9.2
- Docker version 19.03.8

![The Docker Desktop About window print](/assets/dockerdesktop.png?raw=true "Docker Desktop About")

## **Step 1:** Clone the repo and cd into the directory
```
git clone https://github.com/denali49/fabric-ca-k8s.git && cd fabric-ca-k8s
```
NOTE: All of the commands in this tutorial must be run from this directory or they will not work!

## **Step 2:** Install Minikube on your local machine or enable it in Docker Desktop
If you run Docker Desktop, there is a setting that allows you to enable Kubernetes in Docker Desktop.

![Docker Desktop Preferences Screenshot](/assets/k8sdockerenable.png?raw=true "Docker Desktop Preferences")


![Docker Desktop Kubernetes Enable button in Docker Desktop Preferences Screenshot](/assets/dockerdesktopk8s.png?raw=true "Docker Desktop Enable Kubernetes")

If you do not have Docker Desktop with Kubernetes enabled, install minikube.

[Install Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)

In the terminal, run the following command.
```
minikube start
```
Expected output should be similar to this.  ***NOTE: As of this update, the hyperkit driver was depracated so use the flag --driver=virtualbox if you get an error message.*** Check [here](https://kubernetes.io/docs/tasks/tools/install-minikube/) for details on which vitualization drivers are available.

![Screenshot of output from the minikube start command](/assets/minikubestartoutput.png?raw=true "output from minikube start command")

## **Step 3:** Verify minikube installation and that it is running on your local machine
Run the following command in your terminal.
```
kubectl get all
```
Expected output is similar to below, note your ClusterIP will be different.

![Screenshot of output from the kubectl get all command](/assets/minikubeconfirm.png?raw=true "output from `kubectl get all` command")


## **Step 4:** Set up the persistent volume claim and provision storage
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



## **Step 5:** Deploy the fabric-ca-server and perform identity management tasks

Copy the `fabric-ca-server-config.yaml` file into the redis container:

```
kubectl cp $PWD/hyperledger/ redis:/data/redis/
```


Now we are ready to start our Fabric CA Server and interact with it by running the following command in the terminal:
```
kubectl apply -f fabric-ca-deployment.yaml
```
Then make sure it is running by executing:
```
kubectl get pod
```
You can then get the logs of the CA Server by running:

kubectl logs [paste your pod name from the output of the get pod command, paste it here, and hit enter]
So in my case, it was 'kubectl logs fabric-ca-k8s-696566c87f-xz9hx'.  Make sure you copy the name of the fabric-ca-k8s deployment that is 'running' and ***NOT*** the fabric-ca-k8s job that is 'completed'.

Your the last few lines of the log indicating successful server start should be similar to this.
![Screenshot of successful server start](/assets/successlog.png?raw=true "Successful Fabric CA Server start")

Near the top of the log output, you should see the custom values you entered into the fabric-ca-server-config.yaml file.
![Screenshot of new config values after edit](/assets/configeditoutput.png?raw=true "New config values")

Now we are ready to interact with the Fabric CA Server.  First we will 'enroll' the CA admin that was listed in the registry section of the fabric-ca-server-config.yaml file that we modified.  We could have changed it to something else, but note that it was admin:adminpw.  Since this identity was 'registered' automatically by the start up of the server reading from the config file, we simply need to enroll the admin identity.  

For all other idenities we wish to add, we will need to register them first, then enroll them.  For more information on identity management, please see the [Hyperledger Fabric CA Server and Client documentation](https://hyperledger-fabric-ca.readthedocs.io/en/release-1.4/users-guide.html).

Let's get started by getting into the fabric CA container and exporting some environment variables that our client will need.  Find out the pod name of the running pod for the CA by entering the following command:
```
kubectl get pod
```

```
export PATH=${PWD}/bin:${PWD}:$PATH
export FABRIC_CFG_PATH=$PWD/config/
```

```
mkdir -p $PWD/org1/ca/admin
export FABRIC_CA_CLIENT_HOME=$PWD/org1/ca/admin
fabric-ca-client enroll -d -u https://admin:adminpw@159.122.186.71:30102 --caname ca-org1 --tls.certfiles ${PWD}/hyperledger/fabric-ca/tls-cert.pem
```



## **Congratulations!** You have successfully set up your own Hyperledger Fabric Certificate Authority on Kubernetes, modified the Fabric CA Server configuration file, registered and enrolled identities, and modified identities and inspected the result.  You are now ready to explore running your own Fabric Certificate Authority in production systems without using Cryptogen!

Feel free to continue referencing the [Hyperledger Fabric CA documentation](https://hyperledger-fabric-ca.readthedocs.io/en/release-1.4/users-guide.html) and practicing the fabric-ca-client commands against this running instance.  

## **Cleanup**
If you are ready to cleanup, run the following commands:
Type 'exit' to exit the running pod session, then:
```
minikube delete
```


## Scratch



```
kubectl cp $PWD/stuff/ redis:/data/redis/hyperledger/orderer/
```

```
kubectl cp $PWD/stuff/ redis:/data/redis/hyperledger/peer/
```


```
kubectl apply -f fabric-orderer-deployment.yaml
```


```
kubectl apply -f fabric-peer-deployment.yaml
```

```
export CORE_PEER_TLS_ENABLED=true
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=${PWD}/hyperledger/peer/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
export CORE_PEER_ADDRESS=localhost:30112
```
