# contrail-tripleo-howto


# Prepare undercloud installation on the host

## Add stack user
```
sudo useradd stack
sudo passwd stack  # specify a password
echo "stack ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/stack
sudo chmod 0440 /etc/sudoers.d/stack
su - stack
```

## Get undercloud repos
```
sudo curl -L -o /etc/yum.repos.d/delorean-newton.repo https://trunk.rdoproject.org/centos7-newton/current/delorean.repo
sudo curl -L -o /etc/yum.repos.d/delorean-deps-newton.repo http://trunk.rdoproject.org/centos7-newton/delorean-deps.repo
```

## Install undercloud packages
```
sudo yum install -y instack-undercloud
```

## Define under-/overcloud node layout
```
export NODE_CPU=4
export NODE_MEM=16384
export NODE_COUNT=10
export UNDERCLOUD_NODE_CPU=4
export UNDERCLOUD_NODE_MEM=16384
export UNDERCLOUD_NODE_DISK=100
```

## Create and start undercloud VM
```
instack-virt-setup
```

## Retrieve undercloud IP
```
virsh net-dhcp-leases default
```

## Multihost setup (when deploying a single host setup this step can be skipped)
Read the doc (https://github.com/michaelhenkel/tripleo-fabric-ansible) !!!    
Prepare overcloud VMs (this can be done from any host having network access to the KVM hosts)    
```
git clone https://github.com/michaelhenkel/tripleo-fabric-ansible
vi tripleo-fabric-ansible/inventory/hosts # <- add your KVM hosts, specify the interface and path to id_rsa
ansible-playbook -i inventory playbooks/environment_creator.yml
scp /tmp/instackenv.json root@<undercloud VM IP>/home/stack/instackenv_multi.json
```
# Undercloud setup

## Log into undercloud vm
```
ssh root@<undercloud VM IP>
su - stack
```

## Install undercloud
```
openstack undercloud install
```

## Get overcloud images
Go to https://access.redhat.com/downloads/content/220/ver=/rhel---7/7/x86_64/product-software    
Download Ironic Python Agent Image for RHOSP 10.0 RC-1 (ironic-python-agent-10.0-RC-1-2016-11-30.1.tar)    
Download Overcloud Image for RHOSP 10.0 RC-1 (overcloud-full-10.0-RC-1-2016-11-30.1.tar)    
```
mkdir images
mv ironic-python-agent-10.0-RC-1-2016-11-30.1.tar overcloud-full-10.0-RC-1-2016-11-30.1.tar images
cd images
tar xvf ironic-python-agent-10.0-RC-1-2016-11-30.1.tar
tar xvf overcloud-full-10.0-RC-1-2016-11-30.1.tar
cd ..
```
## Upload images to glance
```
openstack overcloud image upload --image-path /home/stack/images/
```

## Import overcloud VMs to ironic 
### Single host
```
openstack baremetal import instackenv.json
```
### Multi host
```
openstack baremetal import instackenv_multi.json
```

## Start vm introspection
```
openstack baremetal introspection bulk start
```

## Get tripleo-heat-templates
```
git clone https://github.com/michaelhenkel/tht -b package_install tripleo-heat-templates
```

## Get tripleo and service puppet modules
```
git clone https://github.com/michaelhenkel/contrail-tripleo -b package_install
cd contrail-tripleo
tar czvf ~/tripleo-heat-templates/environments/contrail/artifacts/puppet-modules.tgz usr
cd ..
```

## Create contrail repo
### Create repo directory
```
cd /var/www/html
mkdir contrail-rhel
cd contrail-rhel
```
### Get contrail install package and extract it
```
wget http://10.84.5.120/github-build/R3.2/LATEST/redhat70/newton/contrail-install-packages_3.2.0.0-9-newton.tgz
tar zxvf contrail-install-packages_3.2.0.0-9-newton.tgz
```

## Upload artifacts to swift
```
cd ~/tripleo-heat-templates/environments/contrail/artifacts
upload-swift-artifacts -f puppet-modules.tgz -f contrail-repo.tgz -f vrouter.tgz -f puppetconf.tgz
```

### Currently there is a bug in the artifacts deploy script so we need this workaround
```
sed -i "s#'##g" ~/.tripleo/environments/deployment-artifacts.yaml
```

# Overcloud Configuration
Adjust number of nodes (do not oversubscribe!), ntp and dns server in ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml    
Adjust subnetting in ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml    
Set rhel registration in ~/tripleo-heat-templates/rhel-registration/environment-rhel-registration.yaml (set rhel_reg_password, rhel_reg_pool_id and rhel_reg_user)    

# Deploy overcloud
```
openstack overcloud deploy --templates tripleo-heat-templates/ \
  -e tripleo-heat-templates/environments/puppet-pacemaker.yaml \
  -e tripleo-heat-templates/environments/contrail/neutron-opencontrail.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-services.yaml \
  -e tripleo-heat-templates/environments/network-isolation.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-net.yaml \
  -e tripleo-heat-templates/environments/network-management.yaml \
  -e tripleo-heat-templates/rhel-registration/environment-rhel-registration.yaml \
  -e tripleo-heat-templates/rhel-registration/rhel-registration-resource-registry.yaml \
  --libvirt-type qemu
```

## Switch from undercloud to overcloud
```
source overcloudrc
```
