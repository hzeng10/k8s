Minikube is a tool that makes it easy to run Kubernetes locally. Minikube runs a single-node Kubernetes cluster inside a Virtual Machine (VM) on laptop for users looking to try out Kubernetes or develop with it day-to-day. We can install Minikube on macOS or Linux.

# 1.1. Installation
### Linux:

1. Install Ubuntu Linux(version: 18.04.3 LTS) via VMware Fusion. (VMware Fusion installation file: VMware-Fusion-11.1.0-13668589.dmg)
2. Follow below installation links to install Minikube on Linux. (Select the Linux tab accordingly)
https://kubernetes.io/docs/tasks/tools/install-minikube/
3. In above installation steps, the Minikube also need virtualization support, please choose Virtual Box as the VM for Minikube cluster. 
### macOS:

1. Follow below installation links to install Minikube on macOS. (Select the macOS tab accordingly)
ttps://kubernetes.io/docs/tasks/tools/install-minikube/
2. In step "Install a Hypervisor", if macOS already installed the VMware Fusion, then we can just use VMware as the hypervisor, but we will need install the VM unified driver.
Install the VMware driver base on VMware Fusion via below commands:
```
sudo chown -R $(whoami) /usr/local/lib/pkgconfig
brew install docker-machine-driver-vmware
```
Make vmware as the default driver:
```
minikube config set vm-driver vmware
```
Or we can also select virtual box as the hypervisor, then download virtual box and install it accordingly.
# 1.2. Useful commands
### Start Minikube
```
minikube start
```

This command will start Minikube, and create a cluster. By default, minikube will allocate 2GB memory which is not enough for large deployment. In order to start Minikube with large memory, we can use below command instead. This command will start minikube with 16GB memory, and allocate 4 cpus.
```
minikube start --memory=16384 --cpus=4
```
### Access Kubernetes dashboard
```
minikube dashboard
```
This command will open Kubernetes dashboard in a browser.

### Access Kubernetes service
```
minikube service <service name>
```
This command will open the Kubernetes service(the default namespace) in a browser. As Minikube does not support load balancer,  we cannot get the external IP of specified service yet, so can use this command to find out the service internal ip and port to access.
We can also access Kubernetes service from other namespace via command minikube service <service name> -n <name space>

### Stop Minikube
```
minikube stop
```
This command will stop Minikube and cluster.

### Delete local Minikube cluster
```
minikube delete
```
This command will delete the local Minikube cluster.
### List the URLs for the services in local cluster
```
minikube service list
```
### Configure default setting of Minikube
```
minikube config set vm-driver vmware
minikube config set memory 16384
minikube config set cpus 4
```
Most of configuration changes are required to delete the previous minikube cluster, and the new setting will be applied to new cluster.

### Enter Minikube VM
```
minikube ssh
```
This command will enter to Minikube VM, VM OS is Linux which includes all k8s/isito and application containers. 

### More commands:
https://kubernetes.io/docs/setup/learning-environment/minikube/
