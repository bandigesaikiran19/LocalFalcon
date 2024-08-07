Firstly Install kind for local setup

1.kind create cluster --name bridge --config cluster.yaml --wait 5m

2.kind get clusters ->it will show your cluster name (ex:-bridge)

#### Install ingress for the local setup 
   3.1 helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
 
   3.2 helm repo update
 
   3.3helm install my-ingress ingress-nginx/ingress-nginx   --set controller.service.type=NodePort   --set controller.service.nodePorts.http=30001   --set controller.service.nodePorts.https=30000
 
##### Now you to edit the coredns 
    4 Type this command to edit configmap coredns file 
        4.1 kubectl edit cm -n kube-system coredns
        it will open the on yaml file in vi it looks like 
           .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
##### Add the this
        file /etc/coredns/my-hlf-domain.com.db my-hlf-domain.com
#####
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
#### Add this lines also 
    my-hlf-domain.com.db: |
    ; my-hlf-domain.com file
    my-hlf-domain.com.            IN     SOA     sns.dns.icann.org. noc.dns.icann.org. 2015082541 7200 3600 1209600 3600
    *.my-hlf-domain.com.          IN      A      179.100.1.2
    (your node ip>kubectl get nodes -owide)
###### 

5.Now you need to edit the depolyment file
    5.1  kubectl edit deployment -n kube-system
    it will open the yaml in vi in that search for volumes add the following    
    5.2 it will be like this 
         volumes:
      - configMap:
          defaultMode: 420
          items:
          - key: Corefile
            path: Corefile
#### Add this lines 
          - key: my-hlf-domain.com.db
            path: my-hlf-domain.com.db
######    
          name: coredns
        name: config-volume

6.Now we need to edit the ingress to pass through the ssl 
    6.1  kubectl edit pods my-ingress-ingress-nginx-controller-656bcfd66-58ggd {pod name }

    it will open the yaml in vi and the line as follow 
      - args:
    - /nginx-ingress-controller
    - --publish-service=$(POD_NAMESPACE)/my-ingress-ingress-nginx-controller
    - --election-id=my-ingress-ingress-nginx-leader
    - --controller-class=k8s.io/ingress-nginx
    - --ingress-class=nginx
    - --configmap=$(POD_NAMESPACE)/my-ingress-ingress-nginx-controller
    - --validating-webhook=:8443
    - --validating-webhook-certificate=/usr/local/certificates/cert
    - --validating-webhook-key=/usr/local/certificates/key
    - --enable-ssl-passthrough

7.   helm install filestore -n filestore helm-charts/filestore/ -f examples/filestore/values.yaml --create-namespace
    7.1  curl http://filestore.my-hlf-domain.com:30001
  ```
  If you're getting the following response then that means your Ingress, Custom DNS etc are working properly.

[![FILESTORE RESPONSE](./images/filestore-response.png)]()


8.Now create the Namespaces for now Im usig orderer org1 org2  initialpeerorg
  8.1 kubectl create ns orderer
  8.2 kubectl -n orderer create secret generic rca-secret --from-literal=user=rca-admin --from-literal=password=rcaComplexPassword
  8.3 kubectl -n orderer create secret generic orderer-secret --from-literal=user=ica-orderer --from-literal=password=icaordererSamplePassword

8.4 helm install root-ca -n orderer helm-charts/fabric-ca -f examples/fabric-ca/root-ca.yaml
8.5 curl https://root-ca.my-hlf-domain.com:30000/cainfo --insecure
8.6 You will get the CA response like below.
8.7 kubectl -n initialpeerorg create secret generic initialpeerorg-secret --from-literal=user=ica-initialpeerorg --from-literal=password=icainitialpeerorgSamplePassword
8.8 kubectl -n org1 create secret generic org1-secret --from-literal=user=ica-org1 --from-literal=password=icaorg1SamplePassword

[![CAINFO RESPONSE](../images/cainfo-curl-response.png)]()
9.Deploy TLSCA
  9.1 kubectl -n orderer create secret generic tlsca-secret --from-literal=user=tls-admin --from-literal=password=TlsComplexPassword
  9.2 helm install tls-ca -n orderer helm-charts/fabric-ca -f examples/fabric-ca/tls-ca.yaml

10.Create ROOTCA identities
  10.1 helm install rootca-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/rootca/rootca-identities.yaml
  10.2 helm install tlsca-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/tlsca/tlsca-identities.yaml
11.Deploy Orderer ICA

  11.1 helm install ica-orderer -n orderer helm-charts/fabric-ca -f examples/fabric-ca/ica-orderer.yaml
  11.2 helm install ica-initialpeerorg -n initialpeerorg helm-charts/fabric-ca -f examples/fabric-ca/ica-initialpeerorg.yaml 
  11.3 helm install orderer-ops -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/orderer/orderer-identities.yaml
  11.4 helm install initialpeerorg-ops -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/identities.yaml

  11.4 helm install cryptogen -n orderer helm-charts/fabric-ops/ -f examples/fabric-ops/orderer/orderer-cryptogen.yaml

  After successful completion of this `cryptogen Job`, you'll see the `Genesisblock` file and `Channel transaction` file in your filestore under your project directory. If your project name is `yourproject`, then your project directory will be created as `/usr/share/nginx/html/yourproject`.

  11.5 helm install orderer -n orderer helm-charts/fabric-orderer/ -f examples/fabric-orderer/orderer.yaml

  11.6 helm install peer -n initialpeerorg helm-charts/fabric-peer/ -f examples/fabric-peer/initialpeerorg/values.yaml
12.Create channel
  12.1. helm install channelcreate -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/channel-create.yaml
  12.2 helm install updateanchorpeer -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/update-anchor-peer.yaml

**Important** :- Before you install chaincode , you need to upload the packaged chaincode  file to the filestore under your project directory. If you don't have any chaincode, you can upload the sample chaincode from our repository `examples/files/basic-chaincode_go_1.0.tar.gz` to `/usr/share/nginx/html/yourproject` path in the filestore server.
If you have your own chaincode, then package it and upload the same to filestore and change the chaincode filename in the [examples/fabric-ops/initialpeerorg/install-chaincode.yaml](./fabric-ops/initialpeerorg/install-chaincode.yaml) values file. This must be changed in all Org's `install-chaincode.yaml` values as well `Values.cc_tar_file`.

13.Install chaincode on Initialpeerorg Peers
  13.1 helm install installchaincode -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/install-chaincode.yaml
14.Deploy Org1 ICA

  1. **Deploy Org1 ICA**

You need to create a kubernetes secret with the one registered with rootca identities registration job and update [examples/fabric-ca/ica-org1.yaml](./fabric-ca/ica-org1.yaml) if you're creating a different secret name. Or create the secret mentioned in this file if you do not want to change it. Once done, apply the following.
```
helm install ica-org1 -n org1 helm-charts/fabric-ca -f examples/fabric-ca/ica-org1.yaml
```
2. **Add Org1 to the network**

Once the `Org1` ICA started successfully, you would need to add this `Org1` to the network. For that, you need to run the following Job in `initialpeerorg`. Comment out the `org2` section from the `Values.organizatons` array in the values file [examples/fabric-ops/initialpeerorg/configure-org-channel.yaml](./fabric-ops/initialpeerorg/configure-org-channel.yaml) for now since we have not deployed the `Org2` yet.
```
helm install configorgchannel -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/configure-org-channel.yaml
```
3. **Create Org1 identities with ica-org1**
```
helm install org1-ca-ops -n org1 helm-charts/fabric-ops/ -f examples/fabric-ops/org1/identities.yaml 
```
4. **Deploy Peers on Org1**
```
helm install peer -n org1 helm-charts/fabric-peer/ -f examples/fabric-peer/org1/values.yaml
```
5. **Install ChainCode on Org2 Peers**
```
helm install installchaincode -n org2 helm-charts/fabric-ops/ -f examples/fabric-ops/org2/install-chaincode.yaml
```
6. **Update Anchor peers of Org2**
```
helm install updateanchorpeer -n org2 helm-charts/fabric-ops/ -f examples/fabric-ops/org2/update-anchor-peer.yaml
```

### Approve & Commit Chain Code

 `Approve & Commit requires` **`collection-config`** `optionally. You can manage it through the variable` **`require_collection_config: "true"`**. `If you make it as true, then you must upload a collection config file to the filestore under your project directory. Don't worry, if you have not made any change in the example deployment till this point then you can use the sample collection config we added` [examples/files/collection-config.json](./files/collection-config.json)

 **Important** :- For additional validation, we have added the uploaded collection config `sha256sum` value in the values files of the approval & commit job. These jobs will fail if the `sha256sum` value of the downloaded `collection-config` and the one provided in the values files are different. So if you're changing the `collection-config`, then kindly update the `sha256sum` value under `Values.collection_config_file_hash`.

1. **Approve ChainCode on Initialpeerorg**

 Ensure that you have updated the Chaincode package ID in [examples/fabric-ops/initialpeerorg/approve-chaincode.yaml](./fabric-ops/initialpeerorg/approve-chaincode.yaml), below are the required fields for updating with your own chaincode details. (This Chaincode package ID update is only required if you use your own chaincode package. If not, simply apply the following helm approval jobs)
- cc_name
- cc_version
- cc_package_id

```
helm install approvechaincode -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/approve-chaincode.yaml
```
2. **Approve ChainCode on Org1**

 Ensure that you have updated the Chaincode package ID in [examples/fabric-ops/org1/approve-chaincode.yaml](./fabric-ops/org1/approve-chaincode.yaml), below are the required fields for updating with your own chaincode details. (This Chaincode package ID update is only required if you use your own chaincode package. If not, simply apply the following helm approval jobs)
- cc_name
- cc_version
- cc_package_id

```
helm install approvechaincode -n org1 helm-charts/fabric-ops/ -f examples/fabric-ops/org1/approve-chaincode.yaml
```

4. **Commit ChainCode on Initialpeerorg**

 Ensure that you have updated the Chaincode package ID in [examples/fabric-ops/initialpeerorg/approve-chaincode.yaml](./fabric-ops/initialpeerorg/approve-chaincode.yaml), below are the required fields for updating with your own chaincode details. (This Chaincode package ID update is only required if you use your own chaincode package. If not, simply apply the following helm commit job)
- cc_name
- cc_version
- cc_package_id

```
helm install commitchaincode -n initialpeerorg helm-charts/fabric-ops/ -f examples/fabric-ops/initialpeerorg/commit-chaincode.yaml
```
 You can verify the commit job log. If everything is success, you will see similar logs at the end of the job.
[![CC commit log](../images/cc-commit-success-log.png)]()

### Deploy fabric-cli tools. (Optional)

Optionally you can deploy a `fabric-cli tool` which has the essential fabric tools for advanced troubleshooting. We have packed it as a helm chart which will do dynamic enrollment for your identities. You don't need seperate values file for this, simply use the existing identity registration value file. You must deploy it per org.

```
helm install cli-tools -n initialpeerorg helm-charts/fabric-tools/ -f examples/fabric-ops/initialpeerorg/identities.yaml
```
 Once deployed, exec into the pod and run `bash /scripts/enroll.sh` and watch the output. All identities will be enrolled and new certs will be available in the respective directory. 
 