Script to clean openstack
=========================

Remove all openstack resources that contain the given string in the name. Example:

.. code-block:: bash

    # deletes substring-server instance, substring-network etc etc
    $ python clean_openstack.py substring

.. code-block:: python
import sys
import time

import novaclient.v2.client as nova_client
import neutronclient.v2_0.client as neutron_client
import os
from neutronclient.common.exceptions import NeutronClientException

import requests.packages.urllib3
requests.packages.urllib3.disable_warnings()

OS_USERNAME = '' or os.environ.get('OS_USERNAME')
OS_PASSWORD = '' or os.environ.get('OS_PASSWORD')
OS_TENANT_NAME = '' or os.environ.get('OS_TENANT_NAME')
OS_AUTH_URL = '' or os.environ.get('OS_AUTH_URL')


class SafeClient:
    def __init__(self, name, actual_client):
        self.name = name
        self.actual_client = actual_client

    @property
    def servers(self):
        return SafeClient('nova.servers', self.actual_client.servers)

    def _args_to_string(self, *args, **kwargs):
        items = map(str, args)
        items.extend([str(a) + '=' + str(b) for a, b in kwargs.iteritems()])
        return '(' + ', '.join(items) + ')'

    def __getattr__(self, item):
        def safe_method(*args, **kwargs):
            try:
                print self.name + '.' + item + self._args_to_string(
                        *args, **kwargs)
                method = getattr(self.actual_client, item)
                result = method(*args, **kwargs)
            except NeutronClientException:
                print 'FAILED'
                return
            print 'OK'
            return result
        return safe_method


class OpenstackCleaner:
    def __init__(self):

        self.nova = nova_client.Client(OS_USERNAME,
                                       OS_PASSWORD,
                                       OS_TENANT_NAME,
                                       OS_AUTH_URL)
        self.neutron = neutron_client.Client(username=OS_USERNAME,
                                             password=OS_PASSWORD,
                                             tenant_name=OS_TENANT_NAME,
                                             auth_url=OS_AUTH_URL)

    def make_safe(self):
        self.neutron = SafeClient('neutron', self.neutron)
        self.nova = SafeClient('nova', self.nova)

    def clean_all(self, pattern):
        self._remove_machines(pattern)
        self._remove_networks(pattern)
        self._remove_routers(pattern)
        self._remove_networks(pattern)
        self._neutron_remove_by_pattern('security_group', pattern)
        self._neutron_remove_generic(
                'floatingip', lambda x: x['port_id'] is None)

    @staticmethod
    def _find_fun(pattern):
        return lambda ent: ent['name'].find(pattern) > -1

    def _neutron_find_generic(self, entity_name, pred):
        list_fun = getattr(self.neutron, 'list_' + entity_name + 's')
        entities_key = entity_name + 's'
        return [x for x in list_fun()[entities_key] if pred(x)]

    def _neutron_find_by_pattern(self, entity_name, pattern):
        return self._neutron_find_generic(
                entity_name, OpenstackCleaner._find_fun(pattern))

    def _neutron_remove_generic(self, entity_name, pred):
        remove_fun = getattr(self.neutron, 'delete_' + entity_name)
        for ent in self._neutron_find_generic(entity_name, pred):
            print 'Removing ' + entity_name + ': ' + ent.get('name', ent['id'])
            remove_fun(ent['id'])

    def _neutron_remove_by_pattern(self, entity_name, pattern):
        self._neutron_remove_generic(
                entity_name, OpenstackCleaner._find_fun(pattern))

    def _remove_networks(self, pattern):
        networks = self._neutron_find_by_pattern('network', pattern)
        if not networks:
            return
        networks_ids = [network['id'] for network in networks]
        print 'Removing ports belonging to networks: ' + \
            ', '.join([x['name'] for x in networks])
        self._neutron_remove_generic(
                'port', lambda port: port['network_id'] in networks_ids)
        for network in networks:
            print 'Removing network ' + network['name']
            self.neutron.delete_network(network['id'])

    def _remove_routers(self, pattern):
        routers = self._neutron_find_by_pattern('router', pattern)
        if not routers:
            return
        subnets = self._neutron_find_by_pattern('subnet', pattern)
        if len(subnets) == 0:
            print "Cannot remove router - appropriate subnet not found."
            sys.exit(1)
        #if len(subnets) > 1:
        #    print "Cannot remove router - more then one subnet found."
        #    sys.exit(1)
        for subnet in subnets:
          print 'Removing routers for subnet ' + subnet['name']
          for router in routers:
              print 'Removing router ' + router['name']
              self.neutron.remove_gateway_router(router['id'])
              self.neutron.remove_interface_router(router['id'],
                                  body={'subnet_id': subnet['id']})
              self.neutron.delete_router(router['id'])

    def _remove_machines(self, pattern):
        found = False
        for server in self.nova.servers.list():
            if server.name.find(pattern) == -1:
                continue
            print "Deleting machine " + server.name
            server.delete()
            found = True

        if not found:
            return

        waiting = True
        while waiting:
            waiting = False
            for server in self.nova.servers.list():
                if server.name.find(pattern) > -1:
                    waiting = True
                    break
            if waiting:
                print "Waiting for " + server.name + " to be deleted."
                time.sleep(1)


if __name__ == '__main__':
    if len(sys.argv) != 2:
        print "Usage: " + sys.argv[0] + \
              " <substring to filter openstack entities>"
        sys.exit(1)

    pattern = sys.argv[1]

    cleaner = OpenstackCleaner()
    cleaner.make_safe()
    cleaner.clean_all(pattern)
