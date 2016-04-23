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

1. Verify Operational Status of Openstack Deployment::

Source the openstack demouser rc file::

    % source /root/userrc

Create neutron resources - networks and router::

    % neutron net-create demo-net
    % neutron subnet-create --name demo-subnet demo-net 10.2.0.0/24
    % neutron router-create demo-router
    % neutron router-interface-add demo-router demo-subnet

Boot VM Instance::

    % nova boot —image cirros-qcow2 --flavor m1.tiny --nic net_id=<demo-net uuid> demoVM

2. Clean up Openstack Environment::

Delete Nova Instance::
	
    % nova delete demoVM

Delete Neutron Resources::

    % neutron net-delete demo-net
    % neutron router-interface-delete demo-router demo-subnet
    % neutron router-delete demo-router 

Disable Neutron Agents from starting via upstart::

    % for service in  l3 metadata dhcp
    do
        echo manual > /etc/init/neutron-${service}-agent.conf.override
        stop neutron-${service}-agent
    done

Delete neutron agents in Neutron::

    % neutron agent-list
    % neutron agent-delete <service_agent_uuid>

The service agents that need to be deleted are l3, metadata, and dhcp.

3. Configure Neutron for Astara Network Orchestration::

In /etc/neutron/neutron.conf, set in the [Default] section::

    core_plugin = astara_neutron.plugins.ml2_neutron_plugin.Ml2Plugin
    service_plugins = astara_neutron.plugins.ml2_neutron_plugin.L3RouterPlugin
    api_extensions_path = /usr/local/lib/python2.7/dist-packages/astara_neutron/extensions/
    notification_driver  = neutron.openstack.common.notifier.rpc_notifier
    
In /etc/neutron/plugins/ml2/ml2_conf.ini, set in the [ml2] section::

    extension_drivers = port_security

In /etc/neutron/plugins/ml2/linux_bridge.ini, set in the [agent] section::
    
    l2_population = True

4. Configure Nova for Astara Orchestration::

In /etc/nova/nova.conf, set in the [DEFAULT] section::

   use_ipv6=True

In /etc/nova/nova.conf, set in the [neutron] section::

   service_metadata_proxy = True

In /etc/nova/policy.json, replace::

   "network:attach_external_network": "rule:admin_api"

   with::

   "network:attach_external_network": "rule:admin_api or role:service"

5. Restart Nova and Neutron Services::

   % restart nova-api
   % restart neutron-server
   % restart neutron-plugin-linuxbridge-agent

6. Create Neutron Resource for Astara Management::

   % neutron net-create astara-mgmt
   % neutron subnet-create --name mgt-subnet astara-mgmt fdca:3ba5:a17a:acda::/64 --ip-version=6 --ipv6_address_mode=slaac --enable_dhcp

  Create a public network::

   % neutron net-create --shared --router:external public
   % neutron subnet-create --name public-subnet public 172.16.0.0/24

7. Install Astara
