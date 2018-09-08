# Raspberry Pi Kubernetes Cluster
>Learning how to create a Kubernetes cluster from scratch with some Raspberry Pis.

* [Hardware Build](#hardware-build)
* [Setup](#setup)
  * [Master Node](#master-node)
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

>__Note:__ These instructions are for Macs.

* Much of this setup is following the [Kubernetes on Raspbian Lite][k8s-raspbian] guide by [Alex Ellis][alex-ellis].

### Master Node

1. Download [Raspbian Stretch Lite][raspbian-download].
2. Flash SD Card with the unzipped raspbian image.
	[Etcher][etcher] is an open-source [Electron][electron] app for flashing OS
	images to SD cards and USB drives.

	```bash
	brew cask install etcher
	```
3. Enable __ssh__ on the SD card:
	```bash
	touch shh /Volumes/boot/
	```


---

## Reference

* [Kubernetes on Raspbian Lite][k8s-raspbian]

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
[k8s-raspbian]:https://gist.github.com/alexellis/fdbc90de7691a1b9edb545c17da2d975
[etcher]:https://etcher.io/
[electron]:https://electronjs.org/
[raspbian-download]:https://downloads.raspberrypi.org/raspbian_lite_latest
