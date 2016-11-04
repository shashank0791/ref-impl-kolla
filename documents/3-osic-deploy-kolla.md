Deploying Openstack Kolla
==========================


Intro
------

This document summarizes the steps to deploy an openstack cloud from the Openstack Kolla documentation. If you want to customize your deployment, please visit the Openstack Kolla documentation website at [OSA](http://docs.openstack.org/developer/kolla/)

#### Environment

By end of this chapter, keeping current configurations you will have an OpenStack environment composed of:
- One deployment host.
- compute hosts.
- controller/infrastructure hosts.
- monitoring hosts.
- network hosts.
- storage hosts.

#### Step 1: Prepare Deployment Host

The first step would be to certain dependecies that would aid in the entire deployment process

__Note:__ If you are in osic-prep-container exit and return back to your host.

1.)  Get information on the newest versions of packages and their dependencies:

```shell
apt-get update
```

2.) For Ubuntu based systems where Docker is used it is recommended to use the latest available LTS kernel. The latest LTS kernel available is the wily kernel (version 4.2). While all kernels should work for Docker, some older kernels may have issues with some of the different Docker backends such as AUFS and OverlayFS. In order to update kernel in Ubuntu 14.04 LTS to 4.2, run:

```shell
apt-get install linux-image-generic-lts-wily -y
reboot
```

3.) Kolla deployment can be done using kolla wrapper which performs almost all functionalities needed to deploy kolla. To install kolla wrapper, execute these commands:

```shell
#Python and python-pip
apt-get install python python-pip python-dev libffi-dev gcc libssl-dev -y
    
#Install Ansible to execute ansible-playbooks
apt-get install -y ansible
    
#Install Dependencies
pip install -r requirements.txt -r test-requirements.txt
pip install -U docker-py

#Install kolla wrapper from source:
cd /opt/kolla
python setup.py install

```

4.) Kolla uses docker containers to deploy openstack services. For this, the docker images need to be pulled into the deployment host and pushed into the docker registry running on deployment host (created in Part 2). Follow these steps to build the images:

```shell
#For purpose of simplicity we will be forcing docker to build openstack images on top of latest ubuntu installed from source with tag version 3.0.0:
kolla-build --registry localhost:4000 --base ubuntu --type source --tag 3.0.0 --push
```

5.) Copy the contents of the /opt/kolla/etc/kolla directory into /etc/. This directory contains the required configuration needed for kolla deployment.

```shell
cp -r /opt/kolla/etc/kolla /etc/
GLOBALS_FILE=/etc/kolla/globals.yml
```

6.) You need to configure the globals.yaml file based on the deployment environment:

```shell
#Change the kolla_base_distro and kolla_install_type to match the type of docker images build in step 4.
sudo sed -i 's/^#kolla_base_distro.*/kolla_base_distro: "ubuntu"/' $GLOBALS_FILE
sudo sed -i 's/^#kolla_install_type.*/kolla_install_type: "source"/' $GLOBALS_FILE

#Use an unused IP on your network as the internal and external vip address.
INTERNAL_IP=""
sudo sed -i 's/^kolla_internal_vip_address.*/kolla_internal_vip_address: "'${INTERNAL_IP}'"/' $GLOBALS_FILE
sudo sed -i 's/^kolla_external_vip_address.*/kolla_external_vip_address: "'${INTERNAL_IP}'"/' $GLOBALS_FILE

#Kolla requires atleast two interfaces: one as network interface for api, storage, cluster and tunnel and other as external port for neutron interface:

sudo sed -i 's/^#network_interface.*/network_interface: "'${FIRST_INTERFACE}'"/g' $GLOBALS_FILE
sudo sed -i 's/^#neutron_external_interface.*/neutron_external_interface: "'${SECOND_INTERFACE}'"/g' $GLOBALS_FILE

#In case of multinode deployment, the deployment host must inform all nodes information about the docker registry:
registry_host=$(echo "`hostname -I | cut -d ' ' -f 1`:4000")
sudo sed -i 's/#docker_registry.*/docker_registry: '${registry_host}'/g' $GLOBALS_FILE
```

#Enable required OpenStack Services

```shell
sudo sed -i 's/#enable_cinder:.*/enable_cinder: "yes"/' $GLOBALS_FILE
sudo sed -i 's/#enable_heat:.*/enable_heat: "yes"/' $GLOBALS_FILE
sudo sed -i 's/#enable_horizon:.*/enable_horizon: "yes"/' $GLOBALS_FILE
sudo sed -i 's/#enable_swift:.*/enable_swift: "yes"/' $GLOBALS_FILE
sudo sed -i 's/#glance_backend_ceph:.*/glance_backend_ceph: "yes"/' $GLOBALS_FILE
sudo sed -i 's/#cinder_backend_ceph:.*/cinder_backend_ceph: "{{ enable_ceph }}"/' $GLOBALS_FILE
```

7.) Generate passwords for individual openstack services:

```shell
#Generate Passwords
kolla-genpwd
#Check passwords.yaml to view passwords.
vi /etc/kolla/passwords.yaml
```

##### Step 2: Bootstrap Servers:

1.) Update Linux Kernel of Target hosts:

Every server in the OSIC RAX Cluster is running two Intel X710 10 GbE NICs. These NICs have not been well tested in Ubuntu and as such the upstream i40e driver in the default 14.04.3 Linux kernel will begin showing issues when you setup VLAN tagged interfaces and bridges.

In order to get around this, you must install an updated Linux kernel.

You can do this by running the following commands:

```shell
cd /root/osic-prep-ansible
ansible -i hosts all -m shell -a "apt-get update; apt-get install -y linux-generic-lts-xenial" --forks 25
```

2.) Reboot Nodes to reflect kernel changes:

Finally, reboot all servers:

```shell
ansible -i hosts all -m shell -a "reboot" --forks 25
```

Once all servers reboot, you can begin installing openstack-ansible.

3.) Bootstrap target host:

```shell
 ansible-playbook -i ansible/inventory/multinode -e @/etc/kolla/globals.yml -e @/etc/kolla/passwords.yml -e CONFIG_DIR=/etc/kolla  -e action=bootstrap-servers /usr/local/share/kolla/ansible/kolla-host.yml --ask-pass
 ```

##### Step 3: Deploy Kolla
1.) Switch to Kolla Directory

```shell
cd /opt/kolla
```

2.) Pre-deployment checks for hosts which includes the port scans and globals.yaml validation:

```shell
kolla-ansible -i ansible/inventory/multinode prechecks
```

3.) Pull all images for containers:

```shell
kolla-ansible -i ansible/inventory/multinode pull
```

4.) Deploy Openstack services:

```shell
kolla-ansible -i ansible/inventory/multinode deploy
```

5.) Create Openstack rc file on deployment node (generated in /etc/kolla):

```shell
kolla-ansible -i ansible/inventory/multinode post-deploy
```