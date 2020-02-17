## How to install and configure GPDB for kubenetes onto minikube in CentOS 7.x

Insgtall KVM Hypervisor supported and docker for minikube
~~~
[root@gpdb-k8s ~]# yum -y install qemu-kvm libvirt libvirt-daemon-kvm docker
[root@gpdb-k8s ~]# systemctl start libvirtd docker
[root@gpdb-k8s ~]# systemctl enable libvirtd docker
~~~

### Configure Kubernetes yum repository
~~~
[root@gpdb-k8s ~]# cat <<'EOF' > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
~~~

Install kubectl and minikube
~~~
[root@gpdb-k8s ~]# yum -y install kubectl
[root@gpdb-k8s ~]# wget https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 -O minikube
[root@gpdb-k8s ~]# wget https://storage.googleapis.com/minikube/releases/latest/docker-machine-driver-kvm2
[root@gpdb-k8s ~]# chmod 755 minikube docker-machine-driver-kvm2
[root@gpdb-k8s ~]# mv minikube docker-machine-driver-kvm2 /usr/local/bin/
~~~

Check version of minikube
~~~
[root@gpdb-k8s ~]# minikube version
minikube version: v1.7.2
commit: 50d543b5fcb0e1c0d7c27b1398a9a9790df09dfb
~~~

Check version of kubectl
~~~
[root@gpdb-k8s ~]# kubectl version -o json
{
  "clientVersion": {
    "major": "1",
    "minor": "17",
    "gitVersion": "v1.17.3",
    "gitCommit": "06ad960bfd03b39c8310aaf92d1e7c12ce618213",
    "gitTreeState": "clean",
    "buildDate": "2020-02-11T18:14:22Z",
    "goVersion": "go1.13.6",
    "compiler": "gc",
    "platform": "linux/amd64"
  }
}
The connection to the server localhost:8080 was refused - did you specify the right host or port?
~~~

Switch sudo usern
~~~
[root@gpdb-k8s ~]# sudo su - jomoon
Last login: Thu Feb 13 11:31:16 KST 2020 on pts/0
~~~

Append user into libvirt group in order to use virsh or virt-manager without having to enter a root password
~~~
[jomoon@gpdb-k8s ~]$ sudo usermod --append --groups libvirt $(whoami)
~~~

Start minikube
~~~
[jomoon@gpdb-k8s ~]$ minikube start --cpus 6 --memory 8192 --vm-driver=kvm2
😄  minikube v1.7.2 on Centos 7.5.1804
✨  Using the kvm2 driver based on user configuration
🔥  Creating kvm2 VM (CPUs=6, Memory=8192MB, Disk=20000MB) ...
🐳  Preparing Kubernetes v1.17.2 on Docker 19.03.5 ...
🚀  Launching Kubernetes ...
🌟  Enabling addons: default-storageclass, storage-provisioner
⌛  Waiting for cluster to come online ...
🏄  Done! kubectl is now configured to use "minikube"
~~~

Check status of minikube
~~~
[jomoon@gpdb-k8s ~]$ minikube status
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
[jomoon@gpdb-k8s ~]$
[jomoon@gpdb-k8s ~]$ minikube service list
|-------------|------------|--------------|-----|
|  NAMESPACE  |    NAME    | TARGET PORT  | URL |
|-------------|------------|--------------|-----|
| default     | kubernetes | No node port |
| kube-system | kube-dns   | No node port |
|-------------|------------|--------------|-----|
~~~


To point shell to minikube's docker-daemon, run:
~~~
[jomoon@gpdb-k8s ~]$ minikube docker-env
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.39.130:2376"
export DOCKER_CERT_PATH="/home/jomoon/.minikube/certs"
export MINIKUBE_ACTIVE_DOCKERD="minikube"
~~~

Check kubernetes cluster info
~~~
[jomoon@gpdb-k8s ~]$ kubectl cluster-info
Kubernetes master is running at https://192.168.39.130:8443
KubeDNS is running at https://192.168.39.130:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
~~~

Check kubenetes node info
~~~
[jomoon@gpdb-k8s ~]$ kubectl get nodes
NAME       STATUS   ROLES    AGE   VERSION
minikube   Ready    master   71m   v1.17.2
~~~

Check if minikube's virtual machine is running
~~~
[jomoon@gpdb-k8s ~]$ sudo virsh list --all
 Id    Name                           State
\----------------------------------------------------
 1     minikube                       running
~~~

Enter minikube's virtual miache via ssh
~~~
[jomoon@gpdb-k8s ~]$ minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)
~~~

Check hostname of minikube
~~~
$ hostname
minikube
~~~

Check processes of docker container
~~~
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
f6da8974b3f4        4689081edb10           "/storage-provisioner"   About an hour ago   Up About an hour                        k8s_storage-provisioner_storage-provisioner_kube-system_66f00996-8cef-4379-96ab-4f263be8bbd5_1
216b8137a9fe        70f311871ae1           "/coredns -conf /etc…"   About an hour ago   Up About an hour                        k8s_coredns_coredns-6955765f44-jjjhh_kube-system_ae418d7f-8786-4b21-ba6f-f161ea9002e1_0
27193425ccdc        70f311871ae1           "/coredns -conf /etc…"   About an hour ago   Up About an hour                        k8s_coredns_coredns-6955765f44-ckk26_kube-system_30ae8074-17aa-4b2d-a04f-df3877d7b785_0
7d4b2bfdc670        cba2a99699bd           "/usr/local/bin/kube…"   About an hour ago   Up About an hour                        k8s_kube-proxy_kube-proxy-lqxfp_kube-system_192f581f-3ab5-4f97-80cf-3ceff04684ea_0
f1808978bcdb        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_coredns-6955765f44-ckk26_kube-system_30ae8074-17aa-4b2d-a04f-df3877d7b785_0
06cf98ecfe13        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_kube-proxy-lqxfp_kube-system_192f581f-3ab5-4f97-80cf-3ceff04684ea_0
c6244f94c95d        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_coredns-6955765f44-jjjhh_kube-system_ae418d7f-8786-4b21-ba6f-f161ea9002e1_0
e027bc70b7fd        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_storage-provisioner_kube-system_66f00996-8cef-4379-96ab-4f263be8bbd5_0
4616ec2e92c6        f52d4c527ef2           "kube-scheduler --au…"   About an hour ago   Up About an hour                        k8s_kube-scheduler_kube-scheduler-minikube_kube-system_9c994ea62a2d8d6f1bb7498f10aa6fcf_0
bdc915b6079f        41ef50a5f06a           "kube-apiserver --ad…"   About an hour ago   Up About an hour                        k8s_kube-apiserver_kube-apiserver-minikube_kube-system_08a4ab161909ddada36e18048cb5b831_0
16a6976e3b96        da5fd66c4068           "kube-controller-man…"   About an hour ago   Up About an hour                        k8s_kube-controller-manager_kube-controller-manager-minikube_kube-system_0ae6cf402f641e9b795a3aebca394220_0
efb7f738fdf9        303ce5db0e90           "etcd --advertise-cl…"   About an hour ago   Up About an hour                        k8s_etcd_etcd-minikube_kube-system_258d78c784be89181916767a7391b680_0
72f715a20aaf        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_kube-scheduler-minikube_kube-system_9c994ea62a2d8d6f1bb7498f10aa6fcf_0
6faecb102614        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_kube-apiserver-minikube_kube-system_08a4ab161909ddada36e18048cb5b831_0
18def2a3eecc        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_kube-controller-manager-minikube_kube-system_0ae6cf402f641e9b795a3aebca394220_0
eaaff187cf2b        k8s.gcr.io/pause:3.1   "/pause"                 About an hour ago   Up About an hour                        k8s_POD_etcd-minikube_kube-system_258d78c784be89181916767a7391b680_0
$ exit
logout
~~~


~~~
Follow these steps to download and install the Greenplum for Kubernetes container images, and install the Greenplum Operator resource.
Download the Greenplum for Kubernetes software from Pivotal Network. The download file has the name: greenplum-for-kubernetes-<version>.tar.gz.
~~~

Go to the directory where you downloaded Greenplum for Kubernetes, and unpack the downloaded software. For example:
~~~
$ cd ~/Downloads
~~~

Extract greenplum-for-kubernetes
~~~
[jomoon@gpdb-k8s Downloads]$ tar xvzf greenplum-for-kubernetes-v1.11.0.tar.gz
greenplum-for-kubernetes-v1.11.0/
greenplum-for-kubernetes-v1.11.0/workspace/
greenplum-for-kubernetes-v1.11.0/workspace/my-gp-instance.yaml
greenplum-for-kubernetes-v1.11.0/workspace/samples/
greenplum-for-kubernetes-v1.11.0/workspace/samples/my-gp-with-gptext-instance.yaml
greenplum-for-kubernetes-v1.11.0/workspace/samples/my-gp-with-pxf-instance.yaml
greenplum-for-kubernetes-v1.11.0/workspace/samples/scripts/
greenplum-for-kubernetes-v1.11.0/workspace/samples/scripts/create_disks.bash
greenplum-for-kubernetes-v1.11.0/workspace/samples/scripts/create_pks_cluster_on_gcp.bash
greenplum-for-kubernetes-v1.11.0/workspace/samples/scripts/regsecret-test.bash
greenplum-for-kubernetes-v1.11.0/workspace/samples/scripts/regsecret-test.yaml
greenplum-for-kubernetes-v1.11.0/README.txt
greenplum-for-kubernetes-v1.11.0/images/
greenplum-for-kubernetes-v1.11.0/images/greenplum-for-kubernetes
greenplum-for-kubernetes-v1.11.0/images/greenplum-for-kubernetes-id
greenplum-for-kubernetes-v1.11.0/images/greenplum-for-kubernetes-tag
greenplum-for-kubernetes-v1.11.0/images/greenplum-operator
greenplum-for-kubernetes-v1.11.0/images/greenplum-operator-id
greenplum-for-kubernetes-v1.11.0/images/greenplum-operator-tag
greenplum-for-kubernetes-v1.11.0/operator/
greenplum-for-kubernetes-v1.11.0/operator/templates/
greenplum-for-kubernetes-v1.11.0/operator/templates/NOTES.txt
greenplum-for-kubernetes-v1.11.0/operator/templates/greenplum-operator-cluster-role-binding.yaml
greenplum-for-kubernetes-v1.11.0/operator/templates/greenplum-operator-cluster-role.yaml
greenplum-for-kubernetes-v1.11.0/operator/templates/greenplum-operator-crds.yaml
greenplum-for-kubernetes-v1.11.0/operator/templates/greenplum-operator-service-account.yaml
greenplum-for-kubernetes-v1.11.0/operator/templates/greenplum-operator.yaml
greenplum-for-kubernetes-v1.11.0/operator/values.yaml
greenplum-for-kubernetes-v1.11.0/operator/Chart.yaml
~~~


Go into the new greenplum-for-kubernetes-<version> directory:
~~~
$ cd ./greenplum-for-kubernetes-*
~~~


~~~
(This step is for Minikube deployments only.) Ensure that the local docker daemon interacts with the Minikube docker container registry:

$ eval $(minikube docker-env)
Note: To undo this docker setting in the current shell, run eval "$(docker-machine env -u)".
~~~

Install epel-release
~~~
[jomoon@gpdb-k8s ~]# sudo yum install epel-release
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * epel: ftp.riken.jp
Resolving Dependencies
--> Running transaction check
---> Package epel-release.noarch 0:7-11 will be updated
---> Package epel-release.noarch 0:7-12 will be an update
--> Finished Dependency Resolution

Dependencies Resolved

============================================================================================
 Package                   Arch                Version              Repository         Size
============================================================================================
Updating:
 epel-release              noarch              7-12                 epel               15 k

Transaction Summary
============================================================================================
Upgrade  1 Package

Total download size: 15 k
Is this ok [y/d/N]: y
Downloading packages:
Delta RPMs disabled because /usr/bin/applydeltarpm not installed.
warning: /var/cache/yum/x86_64/7/epel/packages/epel-release-7-12.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 352c64e5: NOKEY
Public key for epel-release-7-12.noarch.rpm is not installed
epel-release-7-12.noarch.rpm                                         |  15 kB  00:00:01
Retrieving key from file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Importing GPG key 0x352C64E5:
 Userid     : "Fedora EPEL (7) <epel@fedoraproject.org>"
 Fingerprint: 91e9 7d7c 4a5e 96f1 7f3e 888f 6a2f aea2 352c 64e5
 Package    : epel-release-7-11.noarch (installed)
 From       : /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
Is this ok [y/N]: y
Running transaction check
Running transaction test
Transaction test succeeded
Running transaction
  Updating   : epel-release-7-12.noarch                                                 1/2
  Cleanup    : epel-release-7-11.noarch                                                 2/2
  Verifying  : epel-release-7-12.noarch                                                 1/2
  Verifying  : epel-release-7-11.noarch                                                 2/2

Updated:
  epel-release.noarch 0:7-12

Complete!
~~~



~~~
[jomoon@gpdb-k8s ~]$ curl -L https://git.io/get_helm.sh | bash
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0
100  7164  100  7164    0     0   1899      0  0:00:03  0:00:03 --:--:--  7556
Downloading https://get.helm.sh/helm-v2.16.1-linux-amd64.tar.gz
Preparing to install helm and tiller into /usr/local/bin
helm installed into /usr/local/bin/helm
tiller installed into /usr/local/bin/tiller
Run 'helm init' to configure helm.
~~~


~~~
[jomoon@gpdb-k8s ~]$ helm init
Creating /home/jomoon/.helm
Creating /home/jomoon/.helm/repository
Creating /home/jomoon/.helm/repository/cache
Creating /home/jomoon/.helm/repository/local
Creating /home/jomoon/.helm/plugins
Creating /home/jomoon/.helm/starters
Creating /home/jomoon/.helm/cache/archive
Creating /home/jomoon/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/jomoon/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
~~~






~~~
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ docker load -i ./images/greenplum-for-kubernetes
91d23cf5425a: Loading layer 127.3 MB/127.3 MB
f36b28e4310d: Loading layer 11.78 kB/11.78 kB
6cb741cb00b7: Loading layer 15.87 kB/15.87 kB
77008e118980: Loading layer 3.072 kB/3.072 kB
f0445255831e: Loading layer 114.7 MB/114.7 MB
944eb3f9077b: Loading layer 11.31 MB/11.31 MB
64e4d7ef040b: Loading layer 7.226 MB/7.226 MB
a2be1c7a4392: Loading layer 715.3 MB/715.3 MB
678fcb7fdf71: Loading layer 89.92 MB/89.92 MB
e1dfb6491764: Loading layer 109.2 MB/109.2 MB
7390094292fc: Loading layer 67.54 MB/67.54 MB
5561f17afe06: Loading layer 67.54 MB/67.54 MB
a20084de55a7: Loading layer 396.5 MB/396.5 MB
dc0ec19f2c73: Loading layer 255.5 MB/255.5 MB
a130aac90df7: Loading layer 61.24 MB/61.24 MB
dca20834a22f: Loading layer  2.56 kB/2.56 kB
3bec381707b4: Loading layer 3.072 kB/3.072 kB
3ad5e45f3991: Loading layer 109.4 MB/109.4 MB
a93105f10deb: Loading layer  2.56 kB/2.56 kB
ba6c1a75cbeb: Loading layer 3.584 kB/3.584 kB
f65a541dfb28: Loading layer   231 MB/231 MB
cf04b31982f8: Loading layer 713.3 MB/713.3 MB
9f777124eb0d: Loading layer 1.629 MB/1.629 MB
df26ab3c8bda: Loading layer  2.56 kB/2.56 kB
90e77a00f3d6: Loading layer 14.96 MB/14.96 MB
91dd5e4c1a07: Loading layer 144.9 kB/144.9 kB
Loaded image: greenplum-for-kubernetes:v1.11.0
~~~




~~~
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ docker load -i ./images/greenplum-operator
33c58014b5a4: Loading layer  65.5 MB/65.5 MB
a1eabe7eb601: Loading layer 40.65 MB/40.65 MB
511680a9987d: Loading layer 41.01 MB/41.01 MB
Loaded image: greenplum-operator:v1.11.0
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ docker images "greenplum-*"
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
greenplum-operator         v1.11.0             852cafa7ac90        7 weeks ago         269 MB
greenplum-for-kubernetes   v1.11.0             3819d17a577a        7 weeks ago         3.05 GB
~~~



~~~
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ helm install greenplum-operator operator/
Error: This command needs 1 argument: chart name
--> helm 3.0.3 버전 업그레이드 이후에 해결됨
~~~



Every release of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.

Download your desired version
Unpack it (tar -zxvf helm-v3.0.0-linux-amd64.tar.gz)
Find the helm binary in the unpacked directory, and move it to its desired destination (mv linux-amd64/helm /usr/local/bin/helm)
From there, you should be able to run the client and add the stable repo: helm help.

~~~
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ eval $(minikube docker-env)
~~~

~~~
[jomoon@gpdb-k8s greenplum-for-kubernetes-v1.11.0]$ helm install greenplum-operator operator/
NAME: greenplum-operator
LAST DEPLOYED: Thu Feb 13 14:53:23 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
greenplum-operator has been installed.

Please see documentation at:
http://greenplum-kubernetes.docs.pivotal.io/
~~~


~~~
$ kubectl get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/greenplum-operator-667ccc59fd-59pxt   1/1     Running   0          3m22s

NAME                                                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/greenplum-validating-webhook-service-667ccc59fd-59pxt   ClusterIP   10.105.165.4   <none>        443/TCP   3m16s
service/kubernetes                                              ClusterIP   10.96.0.1      <none>        443/TCP   3h7m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/greenplum-operator   1/1     1            1           3m22s

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/greenplum-operator-667ccc59fd   1         1         1       3m22s
~~~



~~~
$ kubectl logs -l app=greenplum-operator
time="2020-02-13T05:53:29Z" level=info msg="started workers"
2020-02-13T05:53:29.649Z	INFO	controller-runtime.metrics	metrics server is starting to listen	{"addr": ":8080"}
2020-02-13T05:53:29.650Z	INFO	controller-runtime.controller	Starting EventSource	{"controller": "greenplumtextservice", "source": "kind source: /, Kind="}
2020-02-13T05:53:29.651Z	INFO	controller-runtime.controller	Starting EventSource	{"controller": "greenplumpxfservice", "source": "kind source: /, Kind="}
2020-02-13T05:53:29.651Z	INFO	setup	starting manager
2020-02-13T05:53:29.651Z	INFO	controller-runtime.manager	starting metrics server	{"path": "/metrics"}
2020-02-13T05:53:29.752Z	INFO	controller-runtime.controller	Starting Controller	{"controller": "greenplumtextservice"}
2020-02-13T05:53:29.752Z	INFO	controller-runtime.controller	Starting Controller	{"controller": "greenplumpxfservice"}
2020-02-13T05:53:29.853Z	INFO	controller-runtime.controller	Starting workers	{"controller": "greenplumtextservice", "worker count": 1}
2020-02-13T05:53:29.853Z	INFO	controller-runtime.controller	Starting workers	{"controller": "greenplumpxfservice", "worker count": 1}
~~~


~~~
$ vi storage-class.yaml
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
~~~

~~~
[jomoon@gpdb-k8s workspace]$ kubectl apply -f storage-class.yaml
storageclass.storage.k8s.io/local-storage created
~~~

~~~
[jomoon@gpdb-k8s workspace]$ vi gpdb-storage-claim.yaml
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: greenplum-local-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 10Gi
~~~

~~~
[jomoon@gpdb-k8s workspace]$  kubectl apply -f gpdb-storage-claim.yaml
~~~

~~~
[jomoon@gpdb-k8s workspace]$ vi gpdb-persistent-storage-for-minikube.yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: greenplum-local-pv-master-0
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: local-storage
  local:
    path: /extra_disk01/greenplum-local-pv-master-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: greenplum-local-pv-master-1
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: local-storage
  local:
    path: /extra_disk01/greenplum-local-pv-master-1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: greenplum-local-pv-segment-a-0
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: local-storage
  local:
    path: /extra_disk01/greenplum-local-pv-segment-a-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: greenplum-local-pv-segment-b-0
spec:
  capacity:
    storage: 1Gi
  accessModes:
  - ReadWriteOnce
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: local-storage
  local:
    path: /extra_disk01/greenplum-local-pv-segment-b-0
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - minikube
~~~

~~~
[jomoon@gpdb-k8s workspace]$ kubectl apply -f gpdb-persistent-storage-for-minikube.yaml
persistentvolume/greenplum-local-pv-master-0 created
persistentvolume/greenplum-local-pv-master-1 created
persistentvolume/greenplum-local-pv-segment-a-0 created
persistentvolume/greenplum-local-pv-segment-b-0 created
~~~



~~~
[jomoon@gpdb-k8s workspace]$ minikube ssh
                         _             _
            _         _ ( )           ( )
  ___ ___  (_)  ___  (_)| |/')  _   _ | |_      __
/' _ ` _ `\| |/' _ `\| || , <  ( ) ( )| '_`\  /'__`\
| ( ) ( ) || || ( ) || || |\`\ | (_) || |_) )(  ___/
(_) (_) (_)(_)(_) (_)(_)(_) (_)`\___/'(_,__/'`\____)

$ sudo mkdir -p /extra_disk01/greenplum-local-pv-segment-b-0
$ sudo mkdir -p /extra_disk01/greenplum-local-pv-segment-a-0
$ sudo mkdir -p /extra_disk01/greenplum-local-pv-master-0
$ sudo mkdir -p /extra_disk01/greenplum-local-pv-master-1
$ exit
logout
~~~


~~~
[jomoon@gpdb-k8s workspace]$ vi gpdb-instances.yaml
apiVersion: "greenplum.pivotal.io/v1"
kind: "GreenplumCluster"
metadata:
  name: my-greenplum
spec:
  masterAndStandby:
    standby: "yes"
    hostBasedAuthentication: |
      # host   all   gpadmin   1.2.3.4/32   trust
      # host   all   gpuser    0.0.0.0/0   md5
    memory: "800Mi"
    cpu: "0.5"
    storage: 1G
    storageClassName: local-storage
    antiAffinity: "no"
  segments:
    primarySegmentCount: 1
    memory: "800Mi"
    cpu: "0.5"
    storage: 1G
    storageClassName: local-storage
    antiAffinity: "no"
    mirros: yes
~~~


~~~
[jomoon@gpdb-k8s workspace]$ kubectl apply -f gpdb-instances.yaml
~~~


~~~
[jomoon@gpdb-k8s workspace]$ kubectl get all
NAME                                      READY   STATUS    RESTARTS   AGE
pod/greenplum-operator-667ccc59fd-txh9v   1/1     Running   0          14m
pod/master-0                              1/1     Running   0          25s
pod/master-1                              0/1     Running   0          25s
pod/segment-a-0                           1/1     Running   0          25s
pod/segment-b-0                           1/1     Running   0          25s

NAME                                                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
service/agent                                                   ClusterIP      None             <none>        22/TCP           28s
service/greenplum                                               LoadBalancer   10.103.112.249   <pending>     5432:31045/TCP   27s
service/greenplum-validating-webhook-service-667ccc59fd-txh9v   ClusterIP      10.101.110.103   <none>        443/TCP          14m
service/kubernetes                                              ClusterIP      10.96.0.1        <none>        443/TCP          30m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/greenplum-operator   1/1     1            1           14m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/greenplum-operator-667ccc59fd   1         1         1       14m

NAME                         READY   AGE
statefulset.apps/master      1/2     26s
statefulset.apps/segment-a   1/1     25s
statefulset.apps/segment-b   1/1     25s

NAME                                                 STATUS    AGE
greenplumcluster.greenplum.pivotal.io/my-greenplum   Pending   30s
~~~



~~~
[jomoon@gpdb-k8s workspace]$ kubectl describe greenplumClusters/my-greenplum
Name:         my-greenplum
Namespace:    default
Labels:       <none>
Annotations:  kubectl.kubernetes.io/last-applied-configuration:
                {"apiVersion":"greenplum.pivotal.io/v1","kind":"GreenplumCluster","metadata":{"annotations":{},"name":"my-greenplum","namespace":"default"...
API Version:  greenplum.pivotal.io/v1
Kind:         GreenplumCluster
Metadata:
  Creation Timestamp:  2020-02-14T01:38:51Z
  Finalizers:
    stopcluster.greenplumcluster.pivotal.io
  Generation:        2
  Resource Version:  3640
  Self Link:         /apis/greenplum.pivotal.io/v1/namespaces/default/greenplumclusters/my-greenplum
  UID:               c8de9e1a-becf-48eb-ba6f-2a90f3f97365
Spec:
  Master And Standby:
    Anti Affinity:              no
    Cpu:                        0.5
    Host Based Authentication:  # host   all   gpadmin   1.2.3.4/32   trust
# host   all   gpuser    0.0.0.0/0   md5

    Memory:              800Mi
    Standby:             yes
    Storage:             1G
    Storage Class Name:  local-storage
  Segments:
    Anti Affinity:          no
    Cpu:                    0.5
    Memory:                 800Mi
    Mirros:                 true
    Primary Segment Count:  1
    Storage:                1G
    Storage Class Name:     local-storage
Status:
  Instance Image:    greenplum-for-kubernetes:v1.11.0
  Operator Version:  greenplum-operator:v1.11.0
  Phase:             Pending
Events:              <none>
~~~



~~~
[jomoon@gpdb-k8s workspace]$ kubectl exec -it master-0 bash

gpadmin@master-0:~$ source /opt/gpdb/greenplum_path.sh
gpadmin@master-0:~$ gpstart
20200214:01:42:28:001434 gpstart:master-0:gpadmin-[INFO]:-Starting gpstart with args:
20200214:01:42:28:001434 gpstart:master-0:gpadmin-[INFO]:-Gathering information and validating the environment...
20200214:01:42:28:001434 gpstart:master-0:gpadmin-[INFO]:-Greenplum Binary Version: 'postgres (Greenplum Database) 5.24.0 build dev'
20200214:01:42:28:001434 gpstart:master-0:gpadmin-[INFO]:-Greenplum Catalog Version: '301705051'
20200214:01:42:28:001434 gpstart:master-0:gpadmin-[INFO]:-Starting Master instance in admin mode
20200214:01:42:29:001434 gpstart:master-0:gpadmin-[INFO]:-Obtaining Greenplum Master catalog information
20200214:01:42:29:001434 gpstart:master-0:gpadmin-[INFO]:-Obtaining Segment details from master...
20200214:01:42:29:001434 gpstart:master-0:gpadmin-[INFO]:-Setting new master era
20200214:01:42:29:001434 gpstart:master-0:gpadmin-[INFO]:-Master Started...
20200214:01:42:29:001434 gpstart:master-0:gpadmin-[INFO]:-Shutting down master
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[WARNING]:-Skipping startup of segment marked down in configuration: on segment-a-0 directory /greenplum/data <<<<<
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:---------------------------
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Master instance parameters
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:---------------------------
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Database                 = template1
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Master Port              = 5432
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Master directory         = /greenplum/data-1
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Timeout                  = 600 seconds
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Master standby start     = On
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:---------------------------------------
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-Segment instances that will be started
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:---------------------------------------
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-   Host          Datadir                  Port    Role
20200214:01:42:31:001434 gpstart:master-0:gpadmin-[INFO]:-   segment-b-0   /greenplum/mirror/data   50000   Primary

Continue with Greenplum instance startup Yy|Nn (default=N):
> y
20200214:01:42:32:001434 gpstart:master-0:gpadmin-[INFO]:-Commencing parallel primary and mirror segment instance startup, please wait...
..
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-Process results...
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-----------------------------------------------------
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-   Successful segment starts                                            = 1
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-   Failed segment starts                                                = 0
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-Skipped segment starts (segments are marked down in configuration)   = 1   <<<<<<<<
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-----------------------------------------------------
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-Successfully started 1 of 1 segment instances, skipped 1 other segments
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-----------------------------------------------------
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-****************************************************************************
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-There are 1 segment(s) marked down in the database
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-To recover from this current state, review usage of the gprecoverseg
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-management utility which will recover failed segment instance databases.
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[WARNING]:-****************************************************************************
20200214:01:42:34:001434 gpstart:master-0:gpadmin-[INFO]:-Starting Master instance master-0 directory /greenplum/data-1
20200214:01:42:35:001434 gpstart:master-0:gpadmin-[INFO]:-Command pg_ctl reports Master master-0 instance active
20200214:01:42:35:001434 gpstart:master-0:gpadmin-[INFO]:-Starting standby master
20200214:01:42:35:001434 gpstart:master-0:gpadmin-[INFO]:-Checking if standby master is running on host: master-1.agent.default.svc.cluster.local  in directory: /greenplum/data-1
20200214:01:42:39:001434 gpstart:master-0:gpadmin-[WARNING]:-Number of segments not attempted to start: 1
20200214:01:42:39:001434 gpstart:master-0:gpadmin-[INFO]:-Check status of database with gpstate utility
gpadmin@master-0:~$ psql
psql (8.3.23)
Type "help" for help.

gpadmin=#
gpadmin=#
gpadmin=# select * from gp_segment_configuration;
 dbid | content | role | preferred_role | mode | status | port  |                 hostname                 |                   address                   | replication_port
------+---------+------+----------------+------+--------+-------+------------------------------------------+---------------------------------------------+------------------
    1 |      -1 | p    | p              | s    | u      |  5432 | master-0                                 | master-0.agent.default.svc.cluster.local    |
    4 |      -1 | m    | m              | s    | u      |  5432 | master-1.agent.default.svc.cluster.local | master-1.agent.default.svc.cluster.local    |
    2 |       0 | m    | p              | s    | d      | 40000 | segment-a-0                              | segment-a-0.agent.default.svc.cluster.local |             6000
    3 |       0 | p    | m              | c    | u      | 50000 | segment-b-0                              | segment-b-0.agent.default.svc.cluster.local |             6001
(4 rows)
~~~

