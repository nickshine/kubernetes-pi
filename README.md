# Raspberry Pi Kubernetes Cluster
>Learning how to create a Kubernetes cluster from scratch with some Raspberry Pis.

* [Hardware Build](#hardware-build)
* [Setup](#setup)
  * [Master Node](#master-node)
  * [Worker Nodes](#worker-nodes)
* [Deploy](#deploy)
* [Reference](#reference)

---

## Hardware Build

This cluster is 3 nodes:

![Pi Cluster](images/pi-cluster-01.jpg)
![Pi Cluster](images/pi-cluster-02.jpg)
![Pi Cluster](images/pi-cluster-03.jpg)

Component | Quantity
--- | ---
[Raspberry Pi 3 Model B+][model-b+] | 2
[Raspberry Pi 3 Model B][model-b] | 1
[Anker PowerPort 6 (60W 6-Port USB Charging Hub)][powerport] | 1
[Black Box USB-Powered 10/100 8-Port Switch][eth-switch] | 1
[32GB Sandisk Ultra microSDHC Card][sandisk] | 3
[Raspberry Pi 3 - 4 Layer Stackable Dob Bone Case][pi-case] | 1
[Cat6 Ethernet Patch Cable][cat6-cable] | 1
[RJ-45 Color Coded Strain Relief Boots][rj45-boots] | 8

### Tools

Component | Quantity
--- | ---
[UbiGear Ethernet Cable Crimper Kit + 100 RJ45][cable-kit] | 1

---

## Setup

>__Note:__ These instructions are for Macs. Much of this setup is following the
[Kubernetes on Raspbian Lite][k8s-raspbian] guide by [Alex Ellis][alex-ellis].

1. Download [Raspbian Stretch Lite][raspbian-download].
2. Flash SD Card with the unzipped raspbian image.
	[Etcher][etcher] is an open-source [Electron][electron] app for flashing OS
	images to SD cards and USB drives.

	```bash
	brew cask install etcher
	```
3. Enable __ssh__ by placing an empty file on the sd card:
	```bash
	touch /Volumes/boot/ssh
	```
	* __Note:__ may need to mount the sd card after flashing. Example: `diskutil mount /dev/disk2s1`.
	* Insert flashed sd card in to pi.
4. SSH in to the Pi directly from Ethernet adapter.
	* [Enable _Internet Sharing_][ssh-mac-ethernet]
	* Raspbian should have [Avahi Daemon][avahi] running, allowing for
	  connection with `raspberrypi.local`:
	```bash
	ping raspberrypi.local
	ssh pi@raspberrypi.local    # default pi password: raspberry
	```
	If the above doesn't work, look for the ethernet adapter (bridge100) inet address:
	```
	ifconfig

	# install nmap and discover ethernet devices
	brew install nmap
	sudo nmap -n -sn 192.168.2.1/24
	ssh pi@192.168.2.2
	```
	* __Note:__ ipaddress may differ

5. Setup Locale and modify hostname to (e.g. `k8s-master`) using `raspi-config` util and reboot.
	```bash
	sudo raspi-config
	```
	__Note:__
	* Select locales with `spacebar` in raspi-config.
	* SSH in to rasspberry pi with `<hostname>.local` now (e.g. `ssh pi@k8s-master.local`).

6. Setup Docker
	```bash
	curl -sSL get.docker.com | sh
	sudo usermod pi -aG docker
	newgrp docker
	```
7. Turn off swap space (required for K8s)
	```bash
	sudo dphys-swapfile swapoff
	sudo dphys-swapfile uninstall
	sudo update-rc.d dphys-swapfile remove
	```
8. Enable __cgroups__ in `/boot/cmdline.txt` and reboot:
	```bash
	bflags="$(head -n1 /boot/cmdline.txt) cgroup_enable=cpuset cgroup_enable=memory"
	echo $bflags | sudo tee /boot/cmdline.txt
	```
	__Note:__ `cgroup_memory=1` might be needed for some pi3 models.

	```bash
	sudo reboot
	```
9. [Install kubeadm][kubeadm]
	```bash
	curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
	echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
	sudo apt-get update
	sudo apt-get install -y kubeadm
	```

### Master Node

>__Note:__ Continue on to the [Worker Nodes](#worker-nodes) setup below for non-master nodes.

1. Initialize master node
	
	```bash
	sudo kubeadm init --token-ttl=0 # takes ~10 mins
	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config
	```
2. Save the generated `join-token` for the other nodes:
	```bash
	# example
	sudo kubeadm join 192.168.3.2:6443 --token l3m1rn.vyo4bpefx51upqzw --discovery-token-ca-cert-hash sha256:1ba58581a3a95c795fd603894c4ff7f7a205004c20cc17e1cbe62a870019d267
	```

3. Verify everything is running (system pods might show `Pending` for a while) and install addons:
	```bash
	kubectl --namespace=kube-system get pods
	```
	* See the [installing addons][install-addons] doc
	* Run `kubectl apply -f [podnetwork].yaml` with one of the addons to deploy it to the cluster.

4. Install a network driver like [Weave Net][weave-net]:
	```bash
	kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
	```

5. Copy cluster `config` to local machine to connect with cluster without `ssh`ing in to the master node:
	```bash
	# from mac
	scp pi@k8s-master.local:~/.kube/config ~/.kube/config-pi
	export KUBECONFIG=$KUBECONFIG:$HOME/.kube/config:$HOME/.kube/config-pi
	kubectl config use-context kubernetes-pi # whatever context you've named for your pi cluster config
	```

### Worker Nodes

Find the other pi's on the ethernet switch with `arp -a`, or `sudo nmap -n -sn 192.168.3.1/24` (ip may differ).

1. Repeat the general [setup](#setup) from above for each worker node.
1. Change hostnames to `k8s-worker-n` via `sudo raspi-config`. After rebooting, it should be possible to `ssh` in without ips:
	```bash
	ssh pi@k8s-worker-1.local
	ssh pi@k8s-worker-2.local
	```
2. Join the nodes to the cluster:
	```bash
	sudo kubeadm join 192.168.3.2:6443 --token l3m1rn.vyo4bpefx51upqzw --discovery-token-ca-cert-hash sha256:1ba58581a3a95c795fd603894c4ff7f7a205004c20cc17e1cbe62a870019d267
	```
3. Verify cluster is set up:
	```bash
	kubectl get nodes
	```
	>__Note:__ run this command from master node.. see step 5 from the [Master Node](#master-node) setup above.

## Deploy

### Deploy the K8s Visualizer

1. Clone the visualizer app serve using `kubectl proxy`:
	```bash
	git clone https://github.com/raghur/gcp-live-k8s-visualizer.git
	kubectl proxy --www=path/to/gcp-live-k8s-visualizer
	```

2. Navigate to http://localhost:8001/static/


### Deploy K8s Dashboard

1. Create the rolebinding listed on the dashboard [access control readme][dashboard-readme]
	```bash
	kubectl apply -f dashboard-admin.yaml
	```

2. Deploy the dashboard:
	```bash
	kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/alternative/kubernetes-dashboard.yaml

	```
3. Allow access to dashboard through master node, by changing the default dashboard service type from `ClusterIP` to `NodePort`
	```bash
	kubectl -n kube-system edit service kubernetes-dashboard
	```

---

## Reference

* [Kubernetes on Raspbian Lite][k8s-raspbian]
* [Headless Raspberry Pi Install][headless-pi]
* [How to SSH into your Raspberry Pi with Mac and Ethernet Cable][ssh-mac-ethernet]
* [Controlling your cluster from machines other than master][k8s-docs-control-outside-master]
* [Organinzing Cluster Access Using Kubeconfig Files][k8s-docs-kubeconfig-files]
* [GCP Live K8s Visualizer][visualizer]

[model-b+]:https://www.raspberrypi.org/products/raspberry-pi-3-model-b-plus/
[model-b]:https://www.raspberrypi.org/products/raspberry-pi-3-model-b/
[powerport]:http://a.co/d/g3Ii5Fr
[eth-switch]:http://a.co/d/9NnN8IS
[sandisk]:http://a.co/d/eC8fl8z
[pi-case]:http://a.co/d/gyPpKsa
[cat6-cable]:http://a.co/d/gOTcmWo
[rj45-boots]: http://a.co/d/ieb0iN0
[cable-kit]:http://a.co/d/jc7bpds
[alex-ellis]:https://gist.github.com/alexellis
[etcher]:https://etcher.io/
[electron]:https://electronjs.org/
[raspbian-download]:https://downloads.raspberrypi.org/raspbian_lite_latest
[avahi]:https://linux.die.net/man/8/avahi-daemon
[kubeadm]:https://kubernetes.io/docs/setup/independent/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl
[install-addons]:https://kubernetes.io/docs/concepts/cluster-administration/addons/
[weave-net]:https://www.weave.works/docs/net/latest/kubernetes/kube-addon/
[visualizer]:https://github.com/raghur/gcp-live-k8s-visualizer
[dashboard-readme]:https://github.com/kubernetes/dashboard/wiki/Access-control#admin-privileges

[k8s-raspbian]:https://gist.github.com/alexellis/fdbc90de7691a1b9edb545c17da2d975
[headless-pi]:https://hackernoon.com/raspberry-pi-headless-install-462ccabd75d0
[ssh-mac-ethernet]:https://medium.com/@tzhenghao/how-to-ssh-into-your-raspberry-pi-with-a-mac-and-ethernet-cable-636a197d055
[k8s-docs-control-outside-master]:https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#optional-controlling-your-cluster-from-machines-other-than-the-master
[k8s-docs-kubeconfig-files]:https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/
