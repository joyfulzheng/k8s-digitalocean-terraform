# k8s-digitalocean-terraform

Deploy your Kubernetes cluster on DigitalOcean using Terraform with one click.

fork from [kubernetes-digitalocean-terraform](https://github.com/kubernetes-digitalocean-terraform/kubernetes-digitalocean-terraform) :)

Different from the original source, we use kubeadm to install k8s components, and we choose Ubuntu system, install ingress-controller by default.

## Requirements

* [DigitalOcean](https://www.digitalocean.com/) account
* DigitalOcean Token [In DO's settings/tokens/new](https://cloud.digitalocean.com/settings/tokens/new)
* [Terraform](https://www.terraform.io/)
* CloudFlare's PKI/TLS toolkit [cfssl](https://github.com/cloudflare/cfssl)

### On Mac

With brew installed, all tools can be installed with

```bash
brew install terraform cfssl kubectl 
```

Do all the following steps from a development machine. It does not matter _where_ it is, as long as it is connected to the internet. This one will be subsequently used to access the cluster via `kubectl`.

## Generate private / public keys

```
ssh-keygen -t rsa -b 4096
```

The system will prompt you for a file path to save the key, we will go with `~/.ssh/id_rsa` in this tutorial.

## Add your public key in the DigitalOcean control panel

[Do it here](https://cloud.digitalocean.com/settings/security). Name it and paste the public key just below `Add SSH Key`.

## Add this key to your SSH agent

```bash
eval `ssh-agent -s`
ssh-add ~/.ssh/id_rsa
```

## Invoke Terraform

We put our DigitalOcean token in the file `./secrets/DO_TOKEN` (this directory is mentioned in `.gitignore`, of course, so we don't leak it)

Then we setup the environment variables (step into `this repository` root). Note that the first variable sets up the *number of workers*

```bash
export TF_VAR_number_of_workers=3
export TF_VAR_do_token=$(cat ./secrets/DO_TOKEN)
export TF_VAR_ssh_fingerprint=$(ssh-keygen -E MD5 -lf ~/.ssh/id_rsa.pub | awk '{print $2}' | sed 's/MD5://g')
```

If you are using an older version of OpenSSH (<6.9), replace the last line with

```bash
export TF_VAR_ssh_fingerprint=$(ssh-keygen -lf ~/.ssh/id_rsa.pub | awk '{print $2}')
```

There is a convenience script for you in `./setup_terraform.sh`. Invoke it as

```bash
./setup_terraform.sh
```

Optionally, you can customize the datacenter *region* via:
```bash
export TF_VAR_do_region=fra1
```
The default region is `nyc3`. You can find a list of available regions from [DigitalOcean](https://developers.digitalocean.com/documentation/v2/#list-all-regions).

After setup, call `terraform apply`

```bash
terraform apply
```

That should do! `kubectl` is configured, so you can just check the nodes (`get no`) and the pods (`get po`).

```bash
$ kubectl get no
NAME          LABELS                               STATUS
X.X.X.X   kubernetes.io/hostname=X.X.X.X   Ready     2m
Y.Y.Y.Y   kubernetes.io/hostname=Y.Y.Y.Y   Ready     2m

$ kubectl --namespace=kube-system get po
NAME                                   READY     STATUS    RESTARTS   AGE
kube-apiserver-X.X.X.X                    1/1       Running   0          13m
kube-controller-manager-X.X.X.X           1/1       Running   0          12m
kube-proxy-X.X.X.X                        1/1       Running   0          12m
kube-proxy-X.X.X.X                        1/1       Running   0          11m
kube-proxy-X.X.X.X                        1/1       Running   0          12m
kube-scheduler-X.X.X.X                    1/1       Running   0          13m
```

You are good to go. Now, we can keep on reading to dive into the specifics.

## Deploy details

We use kubeadm.

### K8s master

The cluster master, running:

* flanneld
* kubelet
* kube-proxy
* kube-apiserver
* kube-controller-manager
* kube-scheduler

#### Cloud config

See the template `00-master.yaml`.

#### Provisions

Once we create this droplet (and get its `IP`), the TLS assets will be created locally (i.e. on the development machine from which we run `terraform`), and put into the directory `secrets` (which, again, is mentioned in `.gitignore`). The TLS assets consist of a server key and certificate for the API server, as well as a client key and certificate to authenticate flanneld and the API server to etcd2.

The TLS assets are copied to appropriate directories on the K8s master using Terraform `file` and `remote-exec` provisioners.

Lastly, we start and enable both `kubelet` and `flanneld`, and finally create the `kube-system` namespace.

### K8s workers

Cluster worker nodes, each running:

* flanneld
* kubelet
* kube-proxy
* docker

#### Cloud config

See the template `01-worker.yaml`.

#### Provisions

For each droplet created, a TLS client key and certificate will be created locally (i.e. on the development machine from which we run `terraform`), and put into the directory `secrets` (which, again, is mentioned in `.gitignore`). 

The TLS assets are then copied to appropriate directories on the worker using Terraform `file` and `remote-exec` provisioners.

Finally, we start and enable `kubelet` and `flanneld`.

### Setup `kubectl`

Use

```
export KUBECONFIG=/path/to/your/k8s-digitalocean-terraform/secrets/admin.conf
```

to configure `kubectl`. 

Test your brand new cluster

```bash
kubectl get nodes
```

You should get something similar to

```
$ kubectl get nodes
NAME          LABELS                               STATUS
X.X.X.X       kubernetes.io/hostname=X.X.X.X       Ready
```

### Thanks

* [kubernetes-digitalocean-terraform/kubernetes-digitalocean-terraform](https://github.com/kubernetes-digitalocean-terraform/kubernetes-digitalocean-terraform)
* [Mosho1/kubernetes-digitalocean-terraform](https://github.com/Mosho1/kubernetes-digitalocean-terraform)

### LICENSE

MIT