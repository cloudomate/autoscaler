# Cluster Autoscaler on G42 Cloud 

## Overview
The cluster autoscaler works with self-built Kubernetes cluster on [g42cloud ECS](https://www.g42cloud.com/intl/en-us/product/as.html#/product/ECS) and
specified [g42cloud Auto Scaling Groups](https://www.g42cloud.com/intl/en-us/product/as.html#/product/AS) 
It runs as a Deployment on a worker node in the cluster. This README will go over some of the necessary steps required 
to get the cluster autoscaler up and running.

## Deployment Steps
### Build Image
#### Environment
1. Download Project

    Get the latest `autoscaler` project and download it to `${GOPATH}/src/k8s.io`. 
    
    This is used for building your image, so the machine you use here should be able to access GCR. Do not use a G42 
    Cloud ECS.

2. Go environment

    Make sure you have Go installed in the above machine.
    
3. Docker environment

    Make sure you have Docker installed in the above machine.
    
#### Build and push the image
Execute the following commands in the directory of `autoscaler/cluster-autoscaler` of the autoscaler project downloaded previously.
The following steps use G42 SoftWare Repository for Container (SWR) as an example registry.

1. Build the `cluster-autoscaler` binary:
    ```
    make build-in-docker
    ```
2. Build the docker image:
    ```
   docker build -t {Image repository address}/{Organization name}/{Image name:tag} .
   ```
    For example:
    ```
    docker build -t swr.ae-ad-1.42cloud.com/{Organization name}/cluster-autoscaler:dev .
    ```
   Follow the `Pull/Push Image` section of `Interactive Walkthroughs` under the SWR console to find the image repository address and organization name,
   and also refer to `My Images` -> `Upload Through Docker Client` in SWR console.
    
3. Login to SWR:
    ```
    docker login -u {Encoded username} -p {Encoded password} {SWR endpoint}
    ```
    
    For example:
    ```
    docker login -u ae-ad-1@ABCD1EFGH2IJ34KLMN -p 1a23bc45678def9g01hi23jk4l56m789nop01q2r3s4t567u89v0w1x23y4z5678 swr.ae-ad-1.g42cloud.com
    ```
   Follow the `Pull/Push Image` section of `Interactive Walkthroughs` under the SWR console to find the encoded username, encoded password and swr endpoint,
   and also refer to `My Images` -> `Upload Through Docker Client` in SWR console.
   
4. Push the docker image to SWR:
    ```
    docker push {Image repository address}/{Organization name}/{Image name:tag}
    ```
   
    For example:
    ```
    docker push swr.ae-ad-1.g42cloud.com/{Organization name}/cluster-autoscaler:dev
    ```
   
5. For the cluster autoscaler to function normally, make sure the `Sharing Type` of the image is `Public`.
    If the cluster has trouble pulling the image, go to SWR console and check whether the `Sharing Type` of the image is 
    `Private`. If it is, click `Edit` button on top right and set the `Sharing Type` to `Public`.  
  

## Build Kubernetes Cluster on ECS   

### 1. Install kubelet, kubeadm and kubectl   


### 2. Install containerd/CRI-O
Please see installation [here](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)


Important Step for containerd, enable systemd cgroup
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup\ =\ false/SystemdCgroup\ =\ true/' /etc/containerd/config.toml
systemctl restart containerd

### 3. Initialize Cluster
Create a Kubeadm.yaml file with the following content:
```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: <API end point ip address]>
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  imagePullPolicy: IfNotPresent
  name: node
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns: {}
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.k8s.io
kind: ClusterConfiguration
kubernetesVersion: 1.24.11
networking:
  dnsDomain: cluster.local
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}

```
Follow this guide for HA setup [here](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md)
Follow this guide to create VIP and map to ECS [here](https://docs.g42cloud.com/en-us/usermanual/vpc/vpc_vip_0001.html)

**note: replace the advertiseAddress to your ECS/VIP ip address**

```bash
kubeadm init --config kubeadm.yaml
```

```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### 4. Install Flannel Network
```bash 
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
### 5. Generate Token
```bash
kubeadm token create --ttl 0
```
Generate a token that never expires. Remember this token since it will be used later.

Get hash key. Remember the key since it will be used later.
```
openssl x509 -in /etc/kubernetes/pki/ca.crt -noout -pubkey | openssl rsa -pubin -outform DER 2>/dev/null | sha256sum | cut -d' ' -f1
```

### 6. Create OS Image with K8S Tools, we have taken ubuntu 20.04 as a an example

- Launch a new ECS ubuntu instance, and install Kubeadm, Kubectl and containerd as described above.
    ```bash
    curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
    apt-get update
    apt-get install -y kubelet=${K8S_VERSION}-00 kubeadm=${K8S_VERSION}-00 kubectl=${K8S_VERSION}-00
    apt-mark hold kubelet kubeadm kubectl
    ```
- Create a script to join the new instance into the k8s cluster.
    ```bash
    cat <<EOF >/etc/init.d/init-k8s.sh
    #!bin/bash
    #chkconfig: 2345 80 90
    swapoff -a
    apt-get install -y kubelet=$K8S_VERSION-00 kubeadm=$K8S_VERSION-00 kubectl=$K8S_VERSION-00
    sudo systemctl enable --now kubelet
    systemctl start containerd
    systemctl enable --now containerd
    tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v$CNI_VERSION.tgz
    mkdir -p /etc/containerd
    containerd config default > /etc/containerd/config.toml
    sed -i 's/SystemdCgroup\ =\ false/SystemdCgroup\ =\ true/' /etc/containerd/config.toml
    systemctl restart containerd
    curl -L https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64 -o ./runc.amd64
    install -m 755 runc.amd64 /usr/local/sbin/runc
    mkdir -p /opt/cni/bin
    curl -L https://github.com/containernetworking/plugins/releases/download/v$CNI_VERSION/cni-plugins-linux-amd64-v$CNI_VERSION.tgz -o cni-plugins-linux-amd64-v$CNI_VERSION.tgz
    kubeadm join --token $TOKEN $API_SERVER_ENDPOINT --discovery-token-ca-cert-hash sha256:$HASHKEY
    EOF
    ```
    Replace the $TOKEN with the one created above.

    Replace the $API_SERVER_ENDPOINT, this could be find in the context file.

    Replace the K8S_VERSION, CNI_VERSION with the version deployed in the previous step 

    ```sh
    cat ~./.kube/config

    # server: https://192.168.0.239:6443
    # the API_SERVER_ENDPOINT is the ip address i.e 192.168.0.239:6443
    ```
- Make this script executable.
    ```
    chmod +x /etc/init.d/init-k8s.sh
    ```
<!--TODO: Remove "previously referred to as master" references from this doc once this terminology is fully removed from k8s-->
- Copy `~/.kube/config` from a control plane (previously referred to as master) node to this ECS `~./kube/config` to setup kubectl on this instance.

- Go to G42 Cloud `Image Management` Service and click on `Create Image`. Select type `System disk image`, select your ECS instance as `Source`, then give it a name and then create.

- Remember this ECS instance ID since it will be used later.

### 7. Create AS Group
- Follow the G42 cloud instruction to create an AS Group.
- Create an AS Configuration, and select private image which we just created. Make sure the AS Configuration with EIP automatically assign.
- While creating the `AS Configuration`, add the following script into `Advanced Settings`.
    ```bash
    #!bin/bash
    IDS=$(ls /var/lib/cloud/instances/)
    for ID in $IDS
    do
    if  ( $ID != $ECS_INSTANCE_ID )
    /usr/bin/kubectl --kubeconfig /etc/kubernetes/kubelet.conf patch node $HOSTNAME -p "{\"spec\":{\"providerID\":\"$ID\"}}"
    fi
    sleep 15
    done
    ```
    Replace the $ECS_INSTANCE_ID with the INSTACE ID recorded in previous step

 - Bind the AS Group with this AS Configuration

### Deploy Cluster Autoscaler
#### Configure credentials
The autoscaler needs a `ServiceAccount` which is granted permissions to the cluster's resources and a `Secret` which 
stores credential (AK/SK in this case) information for authenticating with G42 cloud.
    
Examples of `ServiceAccount` and `Secret` are provided in [examples/cluster-autoscaler-svcaccount.yaml](examples/cluster-autoscaler-svcaccount.yaml)
and [examples/cluster-autoscaler-secret.yaml](examples/cluster-autoscaler-secret.yaml). Modify the Secret 
object yaml file with your credentials.

The following parameters are required in the Secret object yaml file:

- `as-endpoint`
    the endpoint is
    ```
    as.ae-ad-1.g42cloud.com
    ```

- `ecs-endpoint`
    the endpoint is 
    ```
    ecs.ae-ad-1.g42cloud.com
    ```

- `project-id`
    
    Follow this link to find the project-id: [Obtaining a Project ID](https://docs.g42cloud.com/devg/apisign/api-sign-provide-proid.html)

- `access-key` and `secret-key`

    Create and find the G42 cloud access-key and secret-key
required by the Secret object yaml file by referring to [Access Keys](https://docs.g42cloud.com/usermanual/ca/ca_01_0003.html)
and [My Credentials](https://support.g42cloud.com/en-us/usermanual-ca/ca_01_0001.html).


#### Configure deployment
   An example deployment file is provided at [examples/cluster-autoscaler-deployment.yaml](examples/cluster-autoscaler-deployment.yaml). 
   Change the `image` to the image you just pushed, the `cluster-name` to the cluster's id and `nodes` to your
   own configurations of the node pool with format
   ```
   {Minimum number of nodes}:{Maximum number of nodes}:{Node pool name}
   ```
   The above parameters should match the parameters of the AS Group you created.
   
   More configuration options can be added to the cluster autoscaler, such as `scale-down-delay-after-add`, `scale-down-unneeded-time`, etc.
   See available configuration options [here](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#what-are-the-parameters-to-ca).

#### Deploy cluster autoscaler on the cluster
1. Log in to a machine which can manage the cluster with `kubectl`.

    Make sure the machine has kubectl access to the cluster.

2. Create the Service Account:
    ```
    kubectl create -f cluster-autoscaler-svcaccount.yaml
    ```

3. Create the Secret:
    ```
    kubectl create -f cluster-autoscaler-secret.yaml
    ```

4. Create the cluster autoscaler deployment:
    ```
    kubectl create -f cluster-autoscaler-deployment.yaml
    ```

### Testing
Now the cluster autoscaler should be successfully deployed on the cluster. Check it by executing
```
kubectl get pods -n kube-system
```

To see whether it functions correctly, deploy a Service to the cluster, and increase and decrease workload to the
Service. Cluster autoscaler should be able to autoscale the AS Group to accommodate the load.

A simple testing method is like this:
- Create a Deployment and a Service: listening for http request, the deployment should sepcify the resource paramater (quotas) tag

- Create HPA policy for pods to be autoscaled
    * Install [metrics server](https://github.com/kubernetes-sigs/metrics-server) by yourself and create an HPA policy, remember to modify the metrics server yaml (adding these container arguments --kubelet-insecure-tls=true --v=2) by executing something like this:
        ```
        kubectl autoscale deployment [Deployment name] --cpu-percent=10 --min=1 --max=20
        ```  
        The above command creates an HPA policy on the deployment with target average cpu usage of 10%. The number of 
        pods will grow if average cpu usage is above 10%, and will shrink otherwise. The `min` and `max` parameters set
        the minimum and maximum number of pods of this deployment.
- Generate load to the above service

    Example tools for generating workload to an http service are:
    * [Use `hey` command](https://github.com/rakyll/hey) 
    * Use `busybox` image:
        ```
        kubectl run -i --tty busybox --image=busybox:1.28 -- sh

        # send an infinite loop of queries to the service
        while true; do wget -q -O- {Service access address}; done
        ```
    
    Feel free to use other tools which have a similar function.
    
- Wait for pods to be added: as load increases, more pods will be added by HPA

- Wait for nodes to be added: when there's insufficient resource for additional pods, new nodes will be added to the 
cluster by the cluster autoscaler

- Stop the load

- Wait for pods to be removed: as load decreases, pods will be removed by HPA

- Wait for nodes to be removed: as pods being removed from nodes, several nodes will become underutilized or empty, 
and will be removed by the cluster autoscaler

    
## Support & Contact Info

Interested in Cluster Autoscaler on G42 Cloud? Want to talk? Have questions, concerns or great ideas?

Please reach out to us at `k8s_sig@g42cloud.com`.