[![Build Status](https://travis-ci.org/marcindulak/ckad-kind.svg?branch=master)](https://travis-ci.org/marcindulak/ckad-kind)

# Description

A https://github.com/kubernetes-sigs/kind lab environment for the https://training.linuxfoundation.org/training/kubernetes-for-developers/ course.

The environment, consisting of one master and one `k8s` worker node,
allows one to practice all exercises from the course,
except exploring the loadbalancing options available in the cloud.

The setup requires 6GB of RAM (it may work even on 4GB, but slowly) and
20GB of disk storage on a Linux, MacOS or MS Windows laptop.


# Usage

The recommended way of setting up the `kind` cluster is to confine it inside
of a `virtualbox` virtual machine run by `vagrant`.
This virtual machine will serve as the docker host for running `kind`.
The setup can be also performed directly on a "physical" laptop running a Debian/Ubuntu based system.


## Bring up the virtual machine used as `docker` host

1. Install from https://www.virtualbox.org/ and https://www.vagrantup.com/

2. After installation, clone the repository

```sh
git clone https://github.com/marcindulak/ckad-kind
```

3. and download the `vagrant` box

```sh
cd ckad-kind
vagrant box add ubuntu/bionic64
```

4. Extend the size of the root partition in the box, using the [scripts/00-extend-size-of-vmdk-root.sh](scripts/00-extend-size-of-vmdk-root.sh) script.
The vmdk image is located in a subdirectory under `~/.vagrant/boxes`. See https://tuhrig.de/resizing-vagrant-box-disk-space/

5. Bring up the virtual machine with `vagrant`, and ssh into it

```sh
vagrant up
vagrant ssh
cd /vagrant
```

From now on, all commands are to be execute **inside** of the virtual machine ssh session,
**starting** from the `/vagrant` directory.


## Configure the `docker` host

**Note**: skip this whole section if running in the virtual machine started by `vagrant`,
since the configuration steps has been already performed using [Vagrantfile](Vagrantfile).

6. Refresh the repo list

```sh
sudo apt-get update
```

7. Install additional tools

```sh
bash scripts/01-install-tools.sh
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
direnv status
```

8. Install `docker`

```sh
bash scripts/10-install-docker.sh  # (the script makes use of sudo)
# Please logout and login again now to make adding $USER to the docker group effective
bash scripts/11-test-docker.sh
```

9. Allow insecure `docker` registry on localhost.

*WARNING*: the script below restarts `docker`.
After restarting of the `docker` host any `kind` clusters must be recreated
https://github.com/kubernetes-sigs/kind/issues/148


```sh
bash scripts/12-allow-insecure-docker-registry.sh  # (the script makes use of sudo)
```

10. Install `go`

```sh
bash scripts/13-install-go.sh  # (the script makes use of sudo)
source /etc/profile
bash scripts/14-test-go.sh
```


## Bring up the ckad `kind` k8s cluster

11. Install `kind` from source, see https://github.com/kubernetes-sigs/kind

```sh
bash scripts/15-install-kind.sh
```

12. Install `kubectl`

```sh
bash scripts/20-install-kubectl.sh
echo "source <(kubectl completion bash)" >> ~/.bashrc
source ~/.bashrc
```

13. Create the `ckad` kind cluster using a specific k8s version from https://hub.docker.com/r/kindest/node/tags.
The `ckad` cluster is setup under the separate `kind` directory,
where its `.kube/config` file is stored, so it does not overwrite the existing, global `~/.kube/config`.
The local `.envrc` file is used for this purpose, by exporting the `KUBECONFIG` environment variable.

```sh
cd kind
direnv allow
bash ../scripts/21-install-ckad.sh
cd ..
```

14. Install and configure local `docker` registry in a 'kind-registry' `docker` container.
See https://kind.sigs.k8s.io/docs/user/local-registry/ and
the [scripts/24-test-ckad-registry.sh](scripts/24-test-ckad-registry.sh) script for the usage of registry.

The `kind-with-registry.sh` script brings up an extra `kind` cluster `ckad-registry` to be deleted.
Similarly to the `ckad` cluster, the `ckad-registry` cluster is setup under the separate `kind-registry` directory,
where its `.kube/config` file is stored, so it does not overwrite the existing, global `~/.kube/config`.

```sh
cd kind-registry
direnv allow
bash ../scripts/22-install-kind-with-registry-sh.sh
bash ../scripts/23-configure-ckad-registry.sh
bash ../scripts/24-test-ckad-registry.sh
bash ../scripts/25-remove-ckad-registry.sh
cd ..
```

## Configure the k8s cluster

15. Enter the cluster directory. This will load the value of environment variable `KUBECONFIG` from [.envrc](kind/.envrc)

```sh
cd kind
direnv allow
```

16. Install `calico` https://docs.projectcalico.org/v3.11/getting-started/kubernetes/installation/calico
The `calico` default pod CIDR is 192.168.0.0/16 and 10.96.0.0/12 is the default kubeadm service CIDR.
See https://github.com/mauilion/kind-cni-playground and
https://alexbrand.dev/post/creating-a-kind-cluster-with-calico-networking/

```sh
cd manifests
bash ../../scripts/30-install-calico.sh
cd ..
```

17. Install `nginx` ingress. Note that only one `docker` `kind` node container can act as ingress.
In this case the ingress container is run on the k8s master node.
See  https://projectcontour.io/kindly-running-contour/ and
https://kind.sigs.k8s.io/docs/user/ingress/ and https://banzaicloud.com/blog/kind-ingress/.
In the `vagrant` setup the ingress ports 80 and 443 are forwarded to
8080 and 8443 ports, respectively on the `vagrant` host (your laptop).

```sh
cd manifests
bash ../../scripts/31-install-ingress-nginx.sh
bash ../../scripts/32-test-ingress-nginx.sh
cd ..
```

18. Test nodeport functionality (on the fixed `31080` port)

```sh
cd manifests
bash ../../scripts/33-test-nodeport.sh
cd ..
```

19. Install metrics-server

One must modify the `metrics-server/deploy/1.8+/metrics-server-deployment.yaml`
https://github.com/kubernetes-sigs/kind/issues/398#issuecomment-478311167
to allow insecure, InternalIP adresses

```sh
cd manifests
bash ../../scripts/34-install-metrics-server.sh
bash ../../scripts/35-test-metrics-server.sh
cd ..
```


# Cleanup

20. Remove the `kind` cluster and the `docker` registry container

```sh
kind delete cluster --name ckad
docker stop kind-registry
docker rm kind-registry
```

or, if in `vagrant`, just destroy the virtual machine

```sh
vagrant destroy -f
```


# License

Apache-2.0


# Todo
