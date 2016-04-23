Openstack Austin 2016 - Astara Workshop
===================

Openstack Summit - Astara: A Hands-on Installation & Tutorial Workshop

Assumptions
------------

You have a fully operating Openstack environment with, at least: Nova, Keystone, Glance, and Neutron
The OpenStack environment has been verified to be working, and VM instances can be deployed.


Architecture
------------

Openstack Controller with 3 NIC - Management, Tunnel, and External
Openstack Computer with 3 NIC - Management, Tunnel, and External


Installation
------------

All configuration is to be performed on the controller node.

1. Verify Operational Status of Openstack Deployment:

Source the openstack demouser rc file::
```
% source /root/userrc
```

Create neutron resources - networks and router:
```
% neutron net-create demo-net
% neutron subnet-create --name demo-subnet demo-net 10.2.0.0/24
% neutron router-create demo-router
% neutron router-interface-add demo-router demo-subnet
```

Boot VM Instance::
```
% nova boot —image cirros-qcow2 --flavor m1.tiny --nic net_id=<demo-net uuid> demoVM
```

2. Clean up Openstack Environment::

Delete Nova Instance:
```
% nova delete demoVM
```

Delete Neutron Resources::
```
% neutron net-delete demo-net
% neutron router-interface-delete demo-router demo-subnet
% neutron router-delete demo-router 
```
Disable Neutron Agents from starting via upstart::
```
% for service in  l3 metadata dhcp
do
    echo manual > /etc/init/neutron-${service}-agent.conf.override
    stop neutron-${service}-agent
done
```

Delete neutron agents in Neutron::
```
% neutron agent-list
% neutron agent-delete <service_agent_uuid>
```

The service agents that need to be deleted are l3, metadata, and dhcp.

3. Configure Neutron for Astara Network Orchestration::

In /etc/neutron/neutron.conf, set in the [Default] section::
```
    core_plugin = astara_neutron.plugins.ml2_neutron_plugin.Ml2Plugin
    service_plugins = astara_neutron.plugins.ml2_neutron_plugin.L3RouterPlugin
    api_extensions_path = /usr/local/lib/python2.7/dist-packages/astara_neutron/extensions/
    notification_driver  = neutron.openstack.common.notifier.rpc_notifier
```    
In /etc/neutron/plugins/ml2/ml2_conf.ini, set in the [ml2] section::
```
    extension_drivers = port_security
```
In /etc/neutron/plugins/ml2/linux_bridge.ini, set in the [agent] section::
```  
    l2_population = True
```  

4. Configure Nova for Astara Orchestration::

In /etc/nova/nova.conf, set in the [DEFAULT] section::
```  
    use_ipv6=True
```  

In /etc/nova/nova.conf, set in the [neutron] section::
```
    service_metadata_proxy = True
```
In /etc/nova/policy.json, replace::
```
    "network:attach_external_network": "rule:admin_api"
```
   with::
```
    "network:attach_external_network": "rule:admin_api or role:service"
```

5. Restart Nova and Neutron Services::
```
    % restart nova-api
    % restart neutron-server
    % restart neutron-plugin-linuxbridge-agent
```

6. Create Neutron Resource for Astara Management::
```
    % neutron net-create astara-mgmt
    % neutron subnet-create --name mgt-subnet astara-mgmt fdca:3ba5:a17a:acda::/64 --ip-version=6 --ipv6_address_mode=slaac --enable_dhcp
```
Create a public network::
```
    % neutron net-create --shared --router:external public
    % neutron subnet-create --name public-subnet public 172.16.0.0/24
```

7. Install Astara::

Download the astara software from the openstack git repository::
```
    % git clone git://git.openstack.org/openstack/astara
    % git clone git://git.openstack.org/openstack/astara-neutron
    % git clone git://git.openstack.org/openstack/astara-appliance
```

Create Astara user
```
    % useradd --home-dir "/var/lib/astara" --create-home --system --shell /bin/false astara
    % mkdir -p /var/log/astara /var/lib/astara /etc/astara
    % chown -R astara:astara /var/log/astara /var/lib/astara /etc/astara
```

Install the Astara software via pip::
```
    cd ~/astara
    pip install .

    cd ~/astara-neutron
    pip install .
    cd ~
```

8. Configure Astara Orchestrator::

In /etc/astara/orchestrator.ini, set in [oslo_messaging_rabbit] section::
```
rabbit_host = 10.0.1.3
rabbit_userid = guest
rabbit_password = secret
```

set in [database] section::
```
connection = mysql+pymysql://astara:astara@10.0.1.3/astara?charset=utf8
```

set in [keystone_authtoken] section::
```
auth_uri = http://10.0.1.3:5000
project_name = service
password = neutron
username = neutron
auth_url = http://10.0.1.3:35357
auth_plugin = password
```

set in [default] section::
```
management_prefix = fdca:3ba5:a17a:acda::/64
management_net_id = $management_net_uuid
management_subnet_id = $management_subnet_uuid

external_network_id = $public_network_uuid
external_subnet_id = $public_subnet_uuid

interface_driver=astara.common.linux.interface.BridgeInterfaceDriver

provider_rules_path=/etc/astara/provider_rules.json

nova_metadata_ip = 10.0.1.3
neutron_metadata_proxy_shared_secret = openstack
```

9. Build, Upload and Configure Astara Appliance 

Create SSH key for appliance access
```
% ssh-keygen -f /etc/astara/astara_appliance
```

Upload astara appliance to glance
```
% openstack image create astara --public --container-format=bare --disk-format=qcow2 --file /root/astara-appliance/astara.qcow2
```

Create nova flavor for astara appliance usage
```
% openstack flavor create -id 6 --ram 512 --disk 3 --vcpus 1 --public  m1.astara
```

In /etc/astara/orchestrator.ini, set in the [default] section::
```
ssh_public_key = /etc/astara/astara_appliance.pub
```

Define image_uuid in [router] section for appliance
```
image_uuid = $glance_appliance_image_uuid
```

Define instance_flavor for use by appliance
```
instance_flavor = 6
```

10. Create Astara database and tables::

Create Astara DB in mysql::
```
% mysql -u root -pmysql -e 'CREATE DATABASE astara;'
```

Create Service Access ID and permission for DB::
```
% mysql -u root -pmysql -e "GRANT ALL PRIVILEGES ON astara.* TO 'astara'@'localhost' IDENTIFIED BY 'astara';"
% mysql -u root -pmysql -e "GRANT ALL PRIVILEGES ON astara.* TO 'astara'@'%' IDENTIFIED BY 'astara';"
```

Create Astara DB Tables::
```
% astara-dbsync --config-file /etc/astara/orchestrator.ini upgrade
```

11. Create Astara Service and Endpoints::

Create Openstack Service for Astara::
```
% openstack service create --name astara --description "OpenStack Network Orchestrator" astara
```

Create Astara Service Endpoints
```
% openstack endpoint create --region RegionOne astara public  http://<ip_controller>:44250
% openstack endpoint create --region RegionOne astara internal  http://<ip_controller>:44250
% openstack endpoint create --region RegionOne astara admin  http://<ip_controller>44250
```

12. Start Astara Orchestrator Service::

Add Upstart script for astara orchestrator
```
% cd /etc/init/
% wget https://github.com/akanda/astara-summit-tutorial/files/init/astara-orchestrator.conf
```

Add Logrotate script 
```
% cd /etc/logrotate.d/
% wget https://github.com/akanda/astara-summit-tutorial/files/logrotate.d/astara
```

Add sudoers file for astara user
```
% cd /etc/sudoers.d/
% wget https://github.com/akanda/astara-summit-tutorial/files/sudoers.d/astara_sudoers
```

Start Astara Orchestrator process
```
% start astara-orchestrator
```
