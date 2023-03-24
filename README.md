# Kubernetes cluster
A vagrant script for setting up a Kubernetes cluster using Kubeadm

## Pre-requisites

 * **[Vagrant 2.3.4+](https://developer.hashicorp.com/vagrant/downloads)**
 * **[Virtualbox 7.0.6+](https://www.virtualbox.org)**

## How to Run

#Windows, VirtualBox, and Hyper-V
If you encounter an issue with Windows, you will get a blue screen if you attempt to bring up a VirtualBox VM with Hyper-V enabled.

If you wish to use VirtualBox on Windows, you must ensure that Hyper-V is not enabled on Windows. You can turn off the feature with the following Powershell command for Windows 10.

```
Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
```

For Windows 11, you can use an elevated Powershell.

```
bcdedit /set hypervisorlaunchtype off
```

You can also disable Hyper-V in the Windows system settings.

Right click on the Windows button and select ‘Apps and Features’.
Select Turn Windows Features on or off.
Unselect Hyper-V and click OK.

Execute the following vagrant command to start a new Kubernetes cluster, this will start one master and two nodes:

```
vagrant up
```

You can also start invidual machines by vagrant up k8s-head, vagrant up k8s-node-1 and vagrant up k8s-node-2

If more than two nodes are required, you can edit the servers array in the Vagrantfile

```
servers = [
    {
        :name => "k8s-node-3",
        :type => "node",
        :box => "ubuntu/xenial64",
        :box_version => "20180831.0.0",
        :k8s_version => "1.15.0-00",
        :k8s_release => "stable-1.15",
        :eth1 => "192.168.205.13",
        :mem => "2048",
        :cpu => "2"
    }
]
 ```

As you can see above, you can also configure IP address, memory, CPU and K8's Release and Version in the servers array. 

The Current Configuration installs K8's version v1.15.0-00, and it can be changed accordingly.  

##Connect To VM

Check th VM status.
```
vagrant status
Current machine states:

k8s-master                running (virtualbox)
k8s-node-1                running (virtualbox)
k8s-node-2                running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
```

List the Host to Guest Ports.
```
vagrant port k8s-master
The forwarded ports for the machine are listed below. Please note that
these values may differ from values configured in the Vagrantfile if the
provider supports automatic port collision detection and resolution.

    22 (guest) => 2200 (host)
```

Connect to VM using localhost(You can also use Putty or Mobaxterm).

#Note: Default LoginUser/Password: vagrant/vagrant
```
ssh -p 2200 localhost -l vagrant
```

## Clean-up

Execute the following command to remove the virtual machines created for the Kubernetes cluster.
```
vagrant destroy -f
```

You can destroy individual machines by vagrant destroy k8s-node-1 -f

## Licensing

[Apache License, Version 2.0](http://opensource.org/licenses/Apache-2.0).
