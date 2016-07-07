Installing an agent from source
================================

So that you can install an agent from a WIP branch, without building a package.

.. code-block:: yaml

    node_templates:
      vm:
        type: cloudify.openstack.nodes.Server
        interfaces:
          cloudify.interfaces.cloudify_agent:
            create:
              implementation: agent.cloudify_agent.installer.operations.create
              executor: central_deployment_agent
              inputs:
                cloudify_agent:
                  source_url: https://github.com/cloudify-cosmo/cloudify-agent/archive/branch.tar.gz
