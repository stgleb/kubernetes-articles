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
kubeadm join <master-ip>:<master-port> --token <bootstrap-token> --discovery-token-ca-cert-hash sha256:f3182310b048b4df2f8e0de207647bd76bd74e308bde69027817d8abadad2f96
```

CA cert can be found on master node in `/etc/kubernetes/pki/ca.crt` by default.

To bootstrap node we need only two binaries - kubectl and kubelet
Download those binaries from [here](https://kubernetes.io/docs/setup/release/notes/#server-binaries). And unpack them

```
wget https://dl.k8s.io/v1.13.0/kubernetes-server-linux-amd64.tar.gz
tar -xvf kubernetes-server-linux-amd64.tar.gz
cp kubernetes/server/bin/kubelet /usr/bin
cp kubernetes/server/bin/kubectl /usr/bin
```

Don't forget to install docker

```
apt-get update
apt install -y docker.io
```


Create a folder that will store kubernetes related files

```
mkdir /etc/kubernetes
mkdir /etc/kubernetes/manifest
mkdir /etc/kubernetes/pki
```

Put ca.crt from master to /etc/kubernetes/pki
For doing TLS bootstrap kubelet requires a few things

1. Path to kubeconfig that doesnt exist and will be stored there
   after successfull bootstrap `--kubeconfig`
2. A path to bootstrap config `--bootstrap-kubeconfig`

We can either craft bootstrap config manually using `kubectl config` command
or just copy kubeconfig from master node.

```
kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-cluster kubernetes --server='https://<master-host>:<master-port>' --certificate-authority=/etc/kubernetes/pki/ca.crt --embed-certs=true

kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-credentials tls-bootstrap-token-user --token=<bootstrap-token>

kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig set-context tls-bootstrap-token-user@kubernetes --user=tls-bootstrap-token-user --cluster=kubernetes

kubectl config --kubeconfig=/etc/kubernetes/bootstrap-kubeconfig use-context tls-bootstrap-token-user@kubernetes

```


The last step we need to do - create systemd
file in `/etc/systemd/system/kubelet.service` to run kubelet
We enable client certificate rotation which is in Beta right now to
make kubelet update certificate each time it is going to expire.

```
[Service]
Restart=always
ExecStart=/root/kubelet --kubeconfig=/etc/kubernetes/kubeconfig \ 
--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubeconfig \ 
--pod-manifest-path=/etc/kubernetes/manifest \
--feature-gates=RotateKubeletClientCertificate=true \
--rotate-certificates

[Install]
WantedBy=multi-user.target
```

Then reload systemd daemon and enable kubelet service to run on start

```
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
```


References:

   https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/
   https://kubernetes.io/docs/tasks/tls/certificate-rotation/
   https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet/
   https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/
   https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7