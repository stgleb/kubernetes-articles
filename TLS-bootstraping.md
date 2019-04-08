TLS bootstraping is mechanism of obtaining certificates
from API to allow newly created node communicate with master securily.
This feature was delivered as General Availability from kubernetes v. 1.12


Setting up master node for TLS boostrapping can be tricky and has many details
that can be found in official doc [here](https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/#kube-apiserver-configuration).

For purpose of this article we will use kubeadm to provision master node
and than create worker node, set up it manually and connect to master.
How to create single master k8s instructions can be found [here](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/)

Assuming node is created we need a few this from it

   1. master host:port
   2. bootstrap token
   3. CA cert

First two can be taken from kubeadm init output down bellow

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join <master-host>:<master-port> --token <bootstrap-token> \
    --discovery-token-ca-cert-hash sha256:<ca-cert-hash>
```

CA cert can be found on master node in `/etc/kubernetes/pki/ca.crt` by default.

To bootstrap node we need only two binaries - kubectl and kubelet
Download those binaries from [here](https://kubernetes.io/docs/setup/release/notes/#server-binaries). And unpack them

```
wget https://dl.k8s.io/v1.13.4/kubernetes-server-linux-amd64.tar.gz
tar -xvf kubernetes-server-linux-amd64.tar.gz
sudo cp kubernetes/server/bin/kubelet /usr/bin
sudo cp kubernetes/server/bin/kubectl /usr/bin
```

Don't forget to install docker

```
sudo apt-get update
sudo apt install -y docker.io
```


Create a folder that will store kubernetes related files

```
sudo mkdir /etc/kubernetes
sudo mkdir /etc/kubernetes/manifests
sudo mkdir /etc/kubernetes/pki
```

Put ca.crt from master to /etc/kubernetes/pki
For doing TLS bootstrap kubelet requires a few things

1. Path to kubeconfig that doesnt exist and will be stored there
   after successfull bootstrap `--kubeconfig`
2. A path to bootstrap config `--bootstrap-kubeconfig`

We can either craft bootstrap config manually using `kubectl config` command
or just copy kubeconfig from master node.

```
sudo kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-cluster kubernetes --server='https://<master-host>:<master-port>' --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true

sudo kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-credentials tls-bootstrap-token-user --token=<bootstrap-token>

sudo kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-context tls-bootstrap-token-user@kubernetes --user=tls-bootstrap-token-user --cluster=kubernetes

sudo kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig use-context tls-bootstrap-token-user@kubernetes

```


The last step we need to do - create systemd
file in `/etc/systemd/system/kubelet.service` to run kubelet
We enable client certificate rotation which is in Beta right now to
make kubelet update certificate each time it is going to expire.

```
[Service]
Restart=always
ExecStart=/usr/bin/kubelet --kubeconfig=/etc/kubernetes/kubeconfig --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubeconfig --pod-manifest-path=/etc/kubernetes/manifests --feature-gates=RotateKubeletClientCertificate=true --rotate-certificates

[Install]
WantedBy=multi-user.target
```

Then reload systemd daemon and enable kubelet service to run on start

```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```

Finally check that node is up and running in Ready state

```
kubectl get  no

NAME         STATUS   ROLES    AGE   VERSION
instance-1   Ready    master   15m   v1.13.4
instance-2   Ready    <none>   30s   v1.13.4
```

References:

   - https://kubernetes.io/docs/setup/independent/install-kubeadm/
   - https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/
   - https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/
   - https://kubernetes.io/docs/tasks/tls/certificate-rotation/
   - https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
   - https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/
   - https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7