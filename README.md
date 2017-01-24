# contrail-tripleo-howto


# Prepare undercloud installation

Follow documentation on    
https://access.redhat.com/documentation/en/red-hat-openstack-platform/10/paged/director-installation-and-usage/chapter-4-installing-the-undercloud    
and    
https://access.redhat.com/documentation/en/red-hat-openstack-platform/10/paged/director-installation-and-usage/chapter-5-configuring-basic-overcloud-requirements-with-the-cli-tools    
Stop at point 5.8. Do not 'openstack deploy', yet.

## Get tripleo-heat-templates
```
git clone https://github.com/michaelhenkel/tht -b clean tripleo-heat-templates
```

## Get tripleo and service puppet modules
```
git clone https://github.com/michaelhenkel/contrail-tripleo -b clean
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
upload-swift-artifacts -f puppet-modules.tgz
```

# Overcloud Configuration
Adjust number of nodes (do not oversubscribe!), ntp, dns server and other parameters (contrail repo url must match what has been configured in 'Create contrail repo')  in ~/tripleo-heat-templates/environments/contrail/contrail-services.yaml  
Adjust subnetting in ~/tripleo-heat-templates/environments/contrail/contrail-net.yaml    
Set rhel registration in ~/tripleo-heat-templates/rhel-registration/environment-rhel-registration.yaml (set rhel_reg_password, rhel_reg_pool_id and rhel_reg_user)    

# Deploy overcloud
```
openstack overcloud deploy --templates tripleo-heat-templates/ \
  -e tripleo-heat-templates/environments/puppet-pacemaker.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-services.yaml \
  -e tripleo-heat-templates/environments/contrail/neutron-opencontrail.yaml \
  -e tripleo-heat-templates/environments/network-isolation.yaml \
  -e tripleo-heat-templates/environments/contrail/contrail-net.yaml \
  -e tripleo-heat-templates/environments/network-management.yaml \
  -e tripleo-heat-templates/rhel-registration/environment-rhel-registration.yaml \
  -e tripleo-heat-templates/rhel-registration/rhel-registration-resource-registry.yaml \
  -e tripleo-heat-templates/environments/ips-from-pool-all.yaml
```

## Switch from undercloud to overcloud
```
source overcloudrc
```
