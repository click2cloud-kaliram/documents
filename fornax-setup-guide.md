# Test Cluster Setup For Fornax 

This doc describeds how to setup test clusters for the 2021-8-30 Fornax release test, whose test cases are given in the release_testplan.md in the same folder as this doc. 

This test requires four clusters, denoted as A,B,C and D.  A,B,C are kubernetes clusters created using kubeadm, while cluster D is an arktos cluster started by running script arktos-up.sh https://github.com/Click2Cloud-Centaurus/arktos/blob/guide-cni-updates/docs/setup-guide/arktos-with-mizar-cni.md .  These clusters are configured in a hierarchical topology, where Cluster B is an edge cluster to Cluster A, C edge to B, and D edge to C. 

Machine A is referred as the "root operator machine" in these two docs.

**Note: Run the commands in these two docs as a root user**

## Machine Preparation

1. Prepare 4  machines, 16 GB, 100G storage, ubuntu 18.04, for the clusters of A, B, C and D.

2. Open the port of 10000 & 10002 in the security group of of machine A, B and C.

3. Open the port of 6443 in the security group of of machine A, B, C and D.

4. In machine A, B, C, create a Kubernetes cluster following doc https://github.com/click2cloud-kaliram/documents/blob/main/kubernetes_cluster_setup.md   or Offical doc https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/.

5. In machine D, clone the repo https://github.com/Click2Cloud-Centaurus/arktos.git and start an Arktos cluster byt running the script arktos-up.sh.

## Kubeedge Configuration

### Kubeonfig File Preparation


Copy the admin kubeconfig file of cluster A to machine B, the kubecofig file of cluster B to the machine of cluster C, and the kubeconfig file of cluster C to the machine of cluster D.

Copy the kubeconfig files of cluster A, B, C and D to the root operator machine.

### In machine A, B C do the following ###

#   Install Golang

sudo wget https://storage.googleapis.com/golang/go1.15.4.linux-amd64.tar.gz

sudo tar -C /usr/local -xzf go1.15.4.linux-amd64.tar.gz

sudo echo 'export PATH=$PATH:/usr/local/go/bin' >> $HOME/.profile

sudo echo 'export GOPATH=$HOME/gopath' >> $HOME/.profile

source $HOME/.profile




#  Install dependency 

apt install -y make gcc jq 


### In machine A  ###

## Clone a repo of https://github.com/CentaurusInfra/fornax, sync to the branch/commit to test. & Build the binaries of edgecore and cloudcore using the commands

git clone  https://github.com/CentaurusInfra/fornax

cd fornax

make WHAT=cloudcore

make WHAT=edgecore

## config cloudcore 

cp /etc/kubernetes/admin.conf /root/.kube/config

mkdir -p /etc/kubeedge/config

_output/local/bin/cloudcore --minconfig > /etc/kubeedge/config/cloudcore.yaml



# Create Certificate in Node-A  and Copy into B,C,D

# Note down the IP address of machine A, B, C, and D, denotes as IP_A, IP_B, IP_C and IP_D, and run the command:

```

build/tools/certgen.sh genCA IP_A IP_B IP_C IP_D

build/tools/certgen.sh genCertAndKey server IP_A IP_B IP_C IP_D

```
#Example

mkdir -p /etc/kubeedge/ca

mkdir -p /etc/kubeedge/certs

build/tools/certgen.sh genCA 192.168.4.51 192.168.4.52 192.168.4.53 192.168.4.54

build/tools/certgen.sh genCertAndKey server 192.168.4.51 192.168.4.52 192.168.4.53 192.168.4.54





# Install CRDs  In Node A, B, C,

kubectl apply -f build/crds/devices/devices_v1alpha2_device.yaml

kubectl apply -f build/crds/devices/devices_v1alpha2_devicemodel.yaml 

kubectl apply -f build/crds/reliablesyncs/cluster_objectsync_v1alpha1.yaml

kubectl apply -f build/crds/reliablesyncs/objectsync_v1alpha1.yaml 

kubectl apply -f  build/crds/router/router_v1_rule.yaml

kubectl apply -f  build/crds/router/router_v1_ruleEndpoint.yaml

kubectl apply -f build/crds/edgecluster/mission_v1.yaml

kubectl apply -f build/crds/edgecluster/edgecluster_v1.yaml


# Start CloudCore in machine A

_output/local/bin/cloudcore



# In machine B

#Copy Certificate from Machine A

mkdir -p /etc/kubeedge/ca

mkdir -p /etc/kubeedge/certs
 
scp -r node-a-ip:/etc/kubeedge/ca /etc/kubeedge/ca

scp -r node-a-ip:/etc/kubeedge/certs /etc/kubeedge/certs


git clone  https://github.com/CentaurusInfra/fornax

cd fornax

make WHAT=cloudcore

make WHAT=edgecore

```
1)  config cloudcore 
```
cp /etc/kubernetes/admin.conf /root/.kube/config

 mkdir -p /etc/kubeedge/config

 _output/local/bin/cloudcore --minconfig > /etc/kubeedge/config/cloudcore.yaml

```
2) config edgecore
```
cp /etc/kubernetes/admin.conf /root/edgecluster.kubeconfig

 _output/local/bin/edgecore --edgeclusterconfig > /etc/kubeedge/config/edgecore.yaml

```
3)  test cluster config
```
chmod a+x tests/edgecluster/hack/update_edgecore_config.sh

tests/edgecluster/hack/update_edgecore_config.sh [cluster_A_kubeconfig_file]


#Example using scp

scp  nodea-ip:/etc/kubernetes/admin.conf /root/node-a.conf

tests/edgecluster/hack/update_edgecore_config.sh /root/node-a.conf


# Start EdgeCore 

chmod a+x /root/fornax/_output/local/bin/kubectl/vanilla/kubectl

_output/local/bin/edgecore --edgecluster


# open anather terminal & Start CloudCore 

cd fornax

_output/local/bin/cloudcore







### In machine C  ######

#Copy Certificate from Machine A
mkdir -p /etc/kubeedge/ca
mkdir -p /etc/kubeedge/certs
 
scp -r node-a-ip:/etc/kubeedge/ca /etc/kubeedge/ca
scp -r node-a-ip:/etc/kubeedge/certs /etc/kubeedge/certs


git clone  https://github.com/CentaurusInfra/fornax

cd fornax

make WHAT=cloudcore

make WHAT=edgecore

```
1)  config cloudcore 
```
cp /etc/kubernetes/admin.conf /root/.kube/config
mkdir -p /etc/kubeedge/config
_output/local/bin/cloudcore --minconfig > /etc/kubeedge/config/cloudcore.yaml

```
2) config edgecore
```
cp /etc/kubernetes/admin.conf /root/edgecluster.kubeconfig
_output/local/bin/edgecore --edgeclusterconfig > /etc/kubeedge/config/edgecore.yaml

```
3)  test cluster config
```

chmod a+x tests/edgecluster/hack/update_edgecore_config.sh

tests/edgecluster/hack/update_edgecore_config.sh [cluster_B_kubeconfig_file]


#Example using scp

scp node-b:/etc/kubernetes/admin.conf /root/node-b.conf

tests/edgecluster/hack/update_edgecore_config.sh /root/node-b.conf


# Start EdgeCore 
chmod a+x /root/fornax/_output/local/bin/kubectl/vanilla/kubectl

_output/local/bin/edgecore --edgecluster


# open anather terminal & Start CloudCore 

_output/local/bin/cloudcore




### In machine D
 
# arktos cluster started by running script arktos-up.sh 
 
 https://github.com/Click2Cloud-Centaurus/arktos/blob/guide-cni-updates/docs/setup-guide/arktos-with-mizar-cni.md
 
 

#Copy Certificate from Machine A
 
mkdir -p /etc/kubeedge/ca
mkdir -p /etc/kubeedge/certs
 
scp -r node-a-ip:/etc/kubeedge/ca /etc/kubeedge/ca
scp -r node-a-ip:/etc/kubeedge/certs /etc/kubeedge/certs

git clone  https://github.com/CentaurusInfra/fornax

cd fornax

make WHAT=edgecore

```
2. config edgecore
```

cp [Cluster_D_kubeconfig_file] /root/edgecluster.kubeconfig
_output/local/bin/edgecore --edgeclusterconfig > /etc/kubeedge/config/edgecore.yaml

chmod a+x tests/edgecluster/hack/update_edgecore_config.sh

tests/edgecluster/hack/update_edgecore_config.sh [cluster_C_kubeconfig_file]
```

update the /etc/kubeedge/config/edgecore.yaml so the section of spec/clusterd looks like the following:

```
  clusterd:
    ...
    kubeDistro: arktos
    labels:
      "company" : "futurewei"
```

# Start EdgeCore 

```
chmod a+x /root/fornax/_output/local/bin/kubectl/vanilla/kubectl

_output/local/bin/edgecore --edgecluster


#open other terminal  verify that the edge-cluster

kubectl get edgeclusters
 
#For Testing  refer below link
 https://github.com/click2cloud-kaliram/documents/blob/main/release_testplan.md
 
 



