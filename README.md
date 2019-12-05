# k8s-kvm
Upstream Kubernetes via KVM Vagrant and Ansible

KVM is highly efficient allowing for a "real" kubernetes cluster to run on a single machine given a decent bit of CPU and RAM

This installation is targeted for a bare metal install on a machine hooked into a lab or home network.  The master and worker nodes will have ip addresses on your network. 

The cluster can be destroyed and created again easily as the networking configuration, Vagrantfile, and Ansible playbook only need to be configured once.  


# Pre-install requirements

KVM needs to run as root, so go ahead and make yourself root

## Identify a block of contiguous free ip addresses.
One is needed for the master and must end in 0.  Then one for each worker node starting with 1.  In the example I used 192.168.11.120 through 192.168.11.123 for a total of four addresses.

## Install dependencies
```
apt-get install -qy qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker python3 vagrant ansible
```

## Create a bridge
This will depend on the ubuntu distro and how you chose to set up networking.  Call the bridge 'br0' if you can.  An example netplan configuration is

```
network:
    ethernets:
        enp1s0:
            dhcp4: true
    bridges:
        br0:
            dhcp4: true
            interfaces:
                - enp1s0
    version: 2
 ```
 
 # Get Kubernetes running
 
 ## Change the network prefix to the free address block you chose
 Open Vagrantfile with your favorite text editor and edit the following line.  Make sure you choose a network block that's good on your network.
 
 ```
 NETWORK_PREFIX="192.168.11.12"
 
```
 
 ## Edit alloted resources if desired

 Also in Vagrantfile.
 
 The number of worker nodes can be changed by editing the following line.  Total number of nodes will be the number of workers
 selected plus a master.  Two or three is recommended.
 
 ```
 NUM_NODES = 3
 ```

You change the allotted number of CPU cores and memory.  Be sure not to overallocate.

``` 
   config.vm.provider :libvirt do |libvirt|
    libvirt.cpus = 2
    libvirt.memory = 4096
```

## Run Vagrant
```
vagrant up
```
Your virtual machines should be created.  It may take several minutes to download the image the first time

## Configure hosts for ansible

Edit hosts file
Make sure it has the correct ip address for the master starting with the ip address block you chose

Each node is an incrment of 1 from the master.  Add or delete worker nodes as needed depending on how many you chose.

```
[kube-masters]
master1.kube.local ansible_host=192.168.11.120

[kube-nodes]
worker1.kube.local ansible_host=192.168.11.121
worker2.kube.local ansible_host=192.168.11.122
worker3.kube.local ansible_host=192.168.11.123

[ubuntu:children]
kube-masters
kube-nodes
```

## Run ansible

This step takes several minutes even on a powerful host.  Initializing a kubernetes cluster is a non-trivial task.

You can run the tasks seperately **recommended for first time**:
```
ansible-playbook -i hosts bootstrap.yml
ansible-playbook -i hosts kube-masters.yml
ansible-playbook -i hosts kube-workers.yml
```


Or run them all together
```
ansible-playbook -i hosts kubernetes.yml
```

**NOTE: you will have to type "yes" several times as prompted to accept the rsa fingerprint for ssh!** 


## SSH to the master node
```ssh kube@192.168.11.120```

Change the ip address to whatever you made the master.  The default username / password is kube / kubernetes



 ## Add kubernetes network plugin
 
 ```
 kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/v0.11.0/Documentation/kube-flannel.yml
 ```
 
 ## Smoke testing cluster
 
 ```
 kubectl create deployment nginx --image=nginx
 kubectl create service nodeport nginx --tcp=80:80
 kubectl describe service nginx
```

# Rebooting the cluster

## Kill the old cluster

Destroy VM's through Vagrant
```vagrant destroy```

Kill the host ssh key fingerprint cache
```rm ~/.ssh/known_hosts```

## Make the cluster again

Run the vagrant up command

Run ansible again

Install Kubernetes networking

