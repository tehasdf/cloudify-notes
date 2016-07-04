Workflow execution model
====================

.. note::

    I'm not 100% sure on all of this :)


#. Workflow is started from the REST service:
    - https://github.com/cloudify-cosmo/cloudify-manager/blob/master/rest-service/manager_rest/workflow_client.py#L25-L55

#. REST service sends a celery task to the management queue
    - https://github.com/cloudify-cosmo/cloudify-manager/blob/master/rest-service/manager_rest/celery_client.py#L58-L71

#. cloudify.dispatch.dispatch task is run
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/dispatch.py#L568

#. WorkflowHandler is constructed and typically `dispatch_to_subprocess` is called
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/dispatch.py#L105-L180

#. Execution environment is prepared - the directory, and `input.json`, `output` files

#. A subprocess is created and the `cloudify/dispatch.py` file is executed
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/dispatch.py#L580-L658

#. WorkflowHandler is created again, in the subprocess

#. `WorkflowHandler.handle` is called
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/dispatch.py#L500-L512

#. WorkflowContext is created
    - in the interesting case ;) it'll use RemoteCloudifyWorkflowContextHandler as the handler

#. .func is called, this depends on the `task_name` from the context, which is looked up in the deployment task mapping
    - eg `cloudify.plugins.workflows.install`
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/plugins/workflows.py#L23

#. task function calls `graph_mode()` and `execute()` populating the task graph in the ctx
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/plugins/lifecycle.py#L172

#. `CloudifyWorkflowNodeInstance.execute_operation` is called from the workflow func
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/workflows/workflow_context.py#L254
    - (or ..RelationshipInstance)

#. operation is executed (task name is retrieved from operation mapping, node context is sent too)
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/workflows/workflow_context.py#L511
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/workflows/workflow_context.py#L633
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/workflows/workflow_context.py#L733
    - https://github.com/cloudify-cosmo/cloudify-plugins-common/blob/master/cloudify/workflows/tasks.py#L350


#. A task (still with task_target=None?) is sent, and somehow (?) in that task, node context is unpacked
   and a new task with the correct task_target is sent

#. The celery worker with the corresponding target queue (eg. one on an agent machine) receives that task
    - goes through cloudify.dispatch.dispatch again
    - a OperationHandler is created
    - ctx is pushed, .func is called
