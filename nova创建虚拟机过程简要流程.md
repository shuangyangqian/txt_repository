nova部署虚拟机源码调用过程简要分析，关于novaclient的程序处理流程暂时还没有分析。后期如果有时间会进一步分析novaclient的程序执行过程，以及客户端和服务之间的http请求响应关系。

nova/api/openstack/compute/servers.py


```
def create(self, req, body):
    ...
    (instances, resv_id) = self.compute_api.create(context,
                            inst_type,
                            image_uuid,
                            display_name=name,
                            display_description=description,
                            availability_zone=availability_zone,
                            forced_host=host, forced_node=node,
                            metadata=server_dict.get('metadata', {}),
                            admin_password=password,
                            requested_networks=requested_networks,
                            check_server_group_quota=True,
    ...
```

上面self.compute_api=compute.API(skip_policy_check=True)

nova/compute/__init__.py


```
CELL_TYPE_TO_CLS_NAME = {'api': 'nova.compute.cells_api.ComputeCellsAPI',
                         'compute': 'nova.compute.api.API',
                         None: 'nova.compute.api.API',
                        }
...                        
def API(*args, **kwargs):
    class_name = _get_compute_api_class_name()
    return importutils.import_object(class_name, *args, **kwargs)
```

上述分析可知，nova/api/openstack/compute/servers.py中的create方法调用的是nova/compute/cells_api.py中的create()方法。

nova/compute/cells_api.py:class ComputeCellsAPI(compute_api.API)


```
    def create(self, *args, **kwargs):
        """We can use the base functionality, but I left this here just
        for completeness.
        """
        return super(ComputeCellsAPI, self).create(*args, **kwargs)
```

从代码代码中可以看出，其调用了父类的create()方法。
从导包可以看出compute_api为nova.compute.api，所以ComputeCellsAPI继承于nova.compute.api.API。查看nova.compute.api.API中的create()方法。

nova.compute.api.API:


```
    @hooks.add_hook("create_instance")
    def create(self, context, instance_type,
               image_href, kernel_id=None, ramdisk_id=None,...):
...

        return self._create_instance(
                       context, instance_type,
                       image_href, kernel_id, ramdisk_id,...)

```

其返回的是self._create_instance()方法。


```
    def _create_instance(self, context, instance_type,
               image_href, kernel_id, ...):
        """Verify all the input parameters regardless of the provisioning
        strategy being performed and schedule the instance(s) for
        creation.
        """
                self.compute_task_api.build_instances(context,
                instances=instances, image=boot_meta,
                filter_properties=filter_properties,
                admin_password=admin_password,
                injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=False)

        return (instances, reservation_id)

```
上面调用了compute_task_api.build_instances()方法。因为self.compute_task_api = conductor.ComputeTaskAPI()，所以转向conductor中。
nova.conductor.__init__.py:


```
from nova.conductor import api as conductor_api

def ComputeTaskAPI(*args, **kwargs):
    use_local = kwargs.pop('use_local', False)
    if CONF.conductor.use_local or use_local:
        api = conductor_api.LocalComputeTaskAPI
    else:
        api = conductor_api.ComputeTaskAPI
    return api(*args, **kwargs)
```
可以看出调用的API接口是nova.conductor.api。

nova.conductor.api.ComputeTaskAPI:


```
    def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True):
        self.conductor_compute_rpcapi.build_instances(context,
                instances=instances, image=image,
                filter_properties=filter_properties,
                admin_password=admin_password, injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping,
                legacy_bdm=legacy_bdm)
```

上述中self.conductor_compute_rpcapi = rpcapi.ComputeTaskAPI()，转向conductor.rpcapi.ComputeTaskAPI。

nova.conductor.rpcapi.ComputeTaskAPI:


```
    def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping, legacy_bdm=True):
        image_p = jsonutils.to_primitive(image)
        version = '1.10'
        if not self.client.can_send_version(version):
            version = '1.9'
            if 'instance_type' in filter_properties:
                flavor = filter_properties['instance_type']
                flavor_p = objects_base.obj_to_primitive(flavor)
                filter_properties = dict(filter_properties,
                                         instance_type=flavor_p)
        kw = {'instances': instances, 'image': image_p,
               'filter_properties': filter_properties,
               'admin_password': admin_password,
               'injected_files': injected_files,
               'requested_networks': requested_networks,
               'security_groups': security_groups}
        if not self.client.can_send_version(version):
            version = '1.8'
            kw['requested_networks'] = kw['requested_networks'].as_tuples()
        if not self.client.can_send_version('1.7'):
            version = '1.5'
            bdm_p = objects_base.obj_to_primitive(block_device_mapping)
            kw.update({'block_device_mapping': bdm_p,
                       'legacy_bdm': legacy_bdm})

        cctxt = self.client.prepare(version=version)
        cctxt.cast(context, 'build_instances', **kw)
```

这里调用cctxt.cast方法，发布消息，根据消息队列的相关传递方法，可知往下继续调用时转到对应目录下的manager.py模块的对应方法中。此处相关的知识可以参考：
http://www.cnblogs.com/littlebugfish/p/4058007.html
http://blog.csdn.net/gaoxingnengjisuan/article/details/11468061


此处直接转到nova.conductor.manager.py中。
nova.conductor.manager.py:


```
 def build_instances(self, context, instances, image, filter_properties,
            admin_password, injected_files, requested_networks,
            security_groups, block_device_mapping=None, legacy_bdm=True):
        # TODO(ndipanov): Remove block_device_mapping and legacy_bdm in version
        #                 2.0 of the RPC API.
        # TODO(danms): Remove this in version 2.0 of the RPC API
        if (requested_networks and
                not isinstance(requested_networks,
                               objects.NetworkRequestList)):
            requested_networks = objects.NetworkRequestList(
                objects=[objects.NetworkRequest.from_tuple(t)
                         for t in requested_networks])
        # TODO(melwitt): Remove this in version 2.0 of the RPC API
        flavor = filter_properties.get('instance_type')
        if flavor and not isinstance(flavor, objects.Flavor):
            # Code downstream may expect extra_specs to be populated since it
            # is receiving an object, so lookup the flavor to ensure this.
            flavor = objects.Flavor.get_by_id(context, flavor['id'])
            filter_properties = dict(filter_properties, instance_type=flavor)

        request_spec = {}
        try:
            # check retry policy. Rather ugly use of instances[0]...
            # but if we've exceeded max retries... then we really only
            # have a single instance.
            scheduler_utils.populate_retry(
                filter_properties, instances[0].uuid)
            request_spec = scheduler_utils.build_request_spec(
                    context, image, instances)
            hosts = self._schedule_instances(
                    context, request_spec, filter_properties)
        except Exception as exc:
            updates = {'vm_state': vm_states.ERROR, 'task_state': None}
            for instance in instances:
                self._set_vm_state_and_notify(
                    context, instance.uuid, 'build_instances', updates,
                    exc, request_spec)
                self._cleanup_allocated_networks(
                    context, instance, requested_networks)
            return

        for (instance, host) in six.moves.zip(instances, hosts):
            try:
                instance.refresh()
            except (exception.InstanceNotFound,
                    exception.InstanceInfoCacheNotFound):
                LOG.debug('Instance deleted during build', instance=instance)
                continue
            local_filter_props = copy.deepcopy(filter_properties)
            scheduler_utils.populate_filter_properties(local_filter_props,
                host)
            # The block_device_mapping passed from the api doesn't contain
            # instance specific information
            bdms = objects.BlockDeviceMappingList.get_by_instance_uuid(
                    context, instance.uuid)

            self.compute_rpcapi.build_and_run_instance(context,
                    instance=instance, host=host['host'], image=image,
                    request_spec=request_spec,
                    filter_properties=local_filter_props,
                    admin_password=admin_password,
                    injected_files=injected_files,
                    requested_networks=requested_networks,
                    security_groups=security_groups,
                    block_device_mapping=bdms, node=host['nodename'],
                    limits=host['limits'])
```

上述方法中有两个比较重要的步骤，第一个是调度instance：

```
hosts = self._schedule_instances(
                    context, request_spec, filter_properties)
```

```
    def _schedule_instances(self, context, request_spec, filter_properties):
        scheduler_utils.setup_instance_group(context, request_spec,
                                             filter_properties)
        # TODO(sbauza): Hydrate here the object until we modify the
        # scheduler.utils methods to directly use the RequestSpec object
        spec_obj = objects.RequestSpec.from_primitives(
            context, request_spec, filter_properties)
        hosts = self.scheduler_client.select_destinations(context, spec_obj)
        return hosts
```

第二个步骤是self.compute_rpcapi = compute_rpcapi.ComputeAPI()，从导包关系from nova.compute import rpcapi as compute_rpcapi可以看出调用的是nova.compute.rpcapi的接口。

nova.compute.rpcapi.py:


```
   def build_and_run_instance(self, ctxt, instance, host, image, request_spec,
            filter_properties, admin_password=None, injected_files=None,
            requested_networks=None, security_groups=None,
            block_device_mapping=None, node=None, limits=None):

        version = '4.0'
        cctxt = self.client.prepare(server=host, version=version)
        cctxt.cast(ctxt, 'build_and_run_instance', instance=instance,
                image=image, request_spec=request_spec,
                filter_properties=filter_properties,
                admin_password=admin_password,
                injected_files=injected_files,
                requested_networks=requested_networks,
                security_groups=security_groups,
                block_device_mapping=block_device_mapping, node=node,
                limits=limits)
```

由消息队列相关知识和上述类似情况，此处可以直接转到nova.compute.manager.py中。

nova.compute.manager.py.ComputeManager:


```
    def build_and_run_instance(self, context, instance, image, request_spec,
                     filter_properties, admin_password=None,
                     injected_files=None, requested_networks=None,
                     security_groups=None, block_device_mapping=None,
                     node=None, limits=None):

        @utils.synchronized(instance.uuid)
        def _locked_do_build_and_run_instance(*args, **kwargs):
            # NOTE(danms): We grab the semaphore with the instance uuid
            # locked because we could wait in line to build this instance
            # for a while and we want to make sure that nothing else tries
            # to do anything with this instance while we wait.
            with self._build_semaphore:
                self._do_build_and_run_instance(*args, **kwargs)

        # NOTE(danms): We spawn here to return the RPC worker thread back to
        # the pool. Since what follows could take a really long time, we don't
        # want to tie up RPC workers.
        utils.spawn_n(_locked_do_build_and_run_instance,
                      context, instance, image, request_spec,
                      filter_properties, admin_password, injected_files,
                      requested_networks, security_groups,
                      block_device_mapping, node, limits)

```

调用了该类下的_do_build_and_run_instance()方法：


```
    def _do_build_and_run_instance(self, context, instance, image,
            request_spec, filter_properties, admin_password, injected_files,
            requested_networks, security_groups, block_device_mapping,
            node=None, limits=None):

        try:
            LOG.debug('Starting instance...', context=context,
                      instance=instance)
            instance.vm_state = vm_states.BUILDING
            instance.task_state = None
            instance.save(expected_task_state=
                    (task_states.SCHEDULING, None))
        except exception.InstanceNotFound:
            msg = 'Instance disappeared before build.'
            LOG.debug(msg, instance=instance)
            return build_results.FAILED
        except exception.UnexpectedTaskStateError as e:
            LOG.debug(e.format_message(), instance=instance)
            return build_results.FAILED

        # b64 decode the files to inject:
        decoded_files = self._decode_files(injected_files)

        if limits is None:
            limits = {}

        if node is None:
            node = self.driver.get_available_nodes(refresh=True)[0]
            LOG.debug('No node specified, defaulting to %s', node,
                      instance=instance)

        try:
            with timeutils.StopWatch() as timer:
                self._build_and_run_instance(context, instance, image,
                        decoded_files, admin_password, requested_networks,
                        security_groups, block_device_mapping, node, limits,
                        filter_properties)
            LOG.info(_LI('Took %0.2f seconds to build instance.'),
                     timer.elapsed(), instance=instance)
            return build_results.ACTIVE
        except exception.RescheduledException as e:
            retry = filter_properties.get('retry')
            if not retry:
                # no retry information, do not reschedule.
                LOG.debug("Retry info not present, will not reschedule",
                    instance=instance)
                self._cleanup_allocated_networks(context, instance,
                    requested_networks)
                compute_utils.add_instance_fault_from_exc(context,
                        instance, e, sys.exc_info(),
                        fault_message=e.kwargs['reason'])
                self._nil_out_instance_obj_host_and_node(instance)
                self._set_instance_obj_error_state(context, instance,
                                                   clean_task_state=True)
                return build_results.FAILED
            LOG.debug(e.format_message(), instance=instance)
            # This will be used for logging the exception
            retry['exc'] = traceback.format_exception(*sys.exc_info())
            # This will be used for setting the instance fault message
            retry['exc_reason'] = e.kwargs['reason']
            # NOTE(comstud): Deallocate networks if the driver wants
            # us to do so.
            # NOTE(vladikr): SR-IOV ports should be deallocated to
            # allow new sriov pci devices to be allocated on a new host.
            # Otherwise, if devices with pci addresses are already allocated
            # on the destination host, the instance will fail to spawn.
            # info_cache.network_info should be present at this stage.
            if (self.driver.deallocate_networks_on_reschedule(instance) or
                self.deallocate_sriov_ports_on_reschedule(instance)):
                self._cleanup_allocated_networks(context, instance,
                        requested_networks)
            else:
                # NOTE(alex_xu): Network already allocated and we don't
                # want to deallocate them before rescheduling. But we need
                # to cleanup those network resources setup on this host before
                # rescheduling.
                self.network_api.cleanup_instance_network_on_host(
                    context, instance, self.host)

            self._nil_out_instance_obj_host_and_node(instance)
            instance.task_state = task_states.SCHEDULING
            instance.save()

            self.compute_task_api.build_instances(context, [instance],
                    image, filter_properties, admin_password,
                    injected_files, requested_networks, security_groups,
                    block_device_mapping)
            return build_results.RESCHEDULED
        except (exception.InstanceNotFound,
                exception.UnexpectedDeletingTaskStateError):
            msg = 'Instance disappeared during build.'
            LOG.debug(msg, instance=instance)
            self._cleanup_allocated_networks(context, instance,
                    requested_networks)
            return build_results.FAILED
        except exception.BuildAbortException as e:
            LOG.exception(e.format_message(), instance=instance)
            self._cleanup_allocated_networks(context, instance,
                    requested_networks)
            self._cleanup_volumes(context, instance.uuid,
                    block_device_mapping, raise_exc=False)
            compute_utils.add_instance_fault_from_exc(context, instance,
                    e, sys.exc_info())
            self._nil_out_instance_obj_host_and_node(instance)
            self._set_instance_obj_error_state(context, instance,
                                               clean_task_state=True)
            return build_results.FAILED
        except Exception as e:
            # Should not reach here.
            msg = _LE('Unexpected build failure, not rescheduling build.')
            LOG.exception(msg, instance=instance)
            self._cleanup_allocated_networks(context, instance,
                    requested_networks)
            self._cleanup_volumes(context, instance.uuid,
                    block_device_mapping, raise_exc=False)
            compute_utils.add_instance_fault_from_exc(context, instance,
                    e, sys.exc_info())
            self._nil_out_instance_obj_host_and_node(instance)
            self._set_instance_obj_error_state(context, instance,
                                               clean_task_state=True)
            return build_results.FAILED

```

该方法中调用了该类下的_build_and_run_instance()方法。


```
    def _build_and_run_instance(self, context, instance, image, injected_files,
            admin_password, requested_networks, security_groups,
            block_device_mapping, node, limits, filter_properties):

        image_name = image.get('name')
        self._notify_about_instance_usage(context, instance, 'create.start',
                extra_usage_info={'image_name': image_name})
        try:
            rt = self._get_resource_tracker(node)
            with rt.instance_claim(context, instance, limits):
                # NOTE(russellb) It's important that this validation be done
                # *after* the resource tracker instance claim, as that is where
                # the host is set on the instance.
                self._validate_instance_group_policy(context, instance,
                        filter_properties)
                image_meta = objects.ImageMeta.from_dict(image)
                with self._build_resources(context, instance,
                        requested_networks, security_groups, image_meta,
                        block_device_mapping) as resources:
                    instance.vm_state = vm_states.BUILDING
                    instance.task_state = task_states.SPAWNING
                    # NOTE(JoshNang) This also saves the changes to the
                    # instance from _allocate_network_async, as they aren't
                    # saved in that function to prevent races.
                    instance.save(expected_task_state=
                            task_states.BLOCK_DEVICE_MAPPING)
                    block_device_info = resources['block_device_info']
                    network_info = resources['network_info']
                    LOG.debug('Start spawning the instance on the hypervisor.',
                              instance=instance)
                    with timeutils.StopWatch() as timer:
                        self.driver.spawn(context, instance, image_meta,
                                          injected_files, admin_password,
                                          network_info=network_info,
                                          block_device_info=block_device_info)
                    LOG.info(_LI('Took %0.2f seconds to spawn the instance on '
                                 'the hypervisor.'), timer.elapsed(),
                             instance=instance)
        except (exception.InstanceNotFound,
                exception.UnexpectedDeletingTaskStateError) as e:
            with excutils.save_and_reraise_exception():
                self._notify_about_instance_usage(context, instance,
                    'create.end', fault=e)
        except exception.ComputeResourcesUnavailable as e:
            LOG.debug(e.format_message(), instance=instance)
            self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
            raise exception.RescheduledException(
                    instance_uuid=instance.uuid, reason=e.format_message())
        except exception.BuildAbortException as e:
            with excutils.save_and_reraise_exception():
                LOG.debug(e.format_message(), instance=instance)
                self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
        except (exception.FixedIpLimitExceeded,
                exception.NoMoreNetworks, exception.NoMoreFixedIps) as e:
            LOG.warning(_LW('No more network or fixed IP to be allocated'),
                        instance=instance)
            self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
            msg = _('Failed to allocate the network(s) with error %s, '
                    'not rescheduling.') % e.format_message()
            raise exception.BuildAbortException(instance_uuid=instance.uuid,
                    reason=msg)
        except (exception.VirtualInterfaceCreateException,
                exception.VirtualInterfaceMacAddressException) as e:
            LOG.exception(_LE('Failed to allocate network(s)'),
                          instance=instance)
            self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
            msg = _('Failed to allocate the network(s), not rescheduling.')
            raise exception.BuildAbortException(instance_uuid=instance.uuid,
                    reason=msg)
        except (exception.FlavorDiskTooSmall,
                exception.FlavorMemoryTooSmall,
                exception.ImageNotActive,
                exception.ImageUnacceptable,
                exception.InvalidDiskInfo) as e:
            self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
            raise exception.BuildAbortException(instance_uuid=instance.uuid,
                    reason=e.format_message())
        except Exception as e:
            self._notify_about_instance_usage(context, instance,
                    'create.error', fault=e)
            raise exception.RescheduledException(
                    instance_uuid=instance.uuid, reason=six.text_type(e))

        # NOTE(alaski): This is only useful during reschedules, remove it now.
        instance.system_metadata.pop('network_allocated', None)

        # If CONF.default_access_ip_network_name is set, grab the
        # corresponding network and set the access ip values accordingly.
        network_name = CONF.default_access_ip_network_name
        if (network_name and not instance.access_ip_v4 and
                not instance.access_ip_v6):
            # Note that when there are multiple ips to choose from, an
            # arbitrary one will be chosen.
            for vif in network_info:
                if vif['network']['label'] == network_name:
                    for ip in vif.fixed_ips():
                        if not instance.access_ip_v4 and ip['version'] == 4:
                            instance.access_ip_v4 = ip['address']
                        if not instance.access_ip_v6 and ip['version'] == 6:
                            instance.access_ip_v6 = ip['address']
                    break

        self._update_instance_after_spawn(context, instance)

        try:
            instance.save(expected_task_state=task_states.SPAWNING)
        except (exception.InstanceNotFound,
                exception.UnexpectedDeletingTaskStateError) as e:
            with excutils.save_and_reraise_exception():
                self._notify_about_instance_usage(context, instance,
                    'create.end', fault=e)

        self._update_scheduler_instance_info(context, instance)
        self._notify_about_instance_usage(context, instance, 'create.end',
                extra_usage_info={'message': _('Success')},
                network_info=network_info)

```

上述函数中最主要的就是：

```
self.driver.spawn(context, instance, image_meta,
                                          injected_files, admin_password,
                                          network_info=network_info,
                                          block_device_info=block_device_info)
```

nova.compute.manager.ComputeManager.__init__:
```
   self.driver = driver.load_compute_driver(self.virtapi, compute_driver)
```
此处的driver为nova.virt.driver.py。

nova.virt.driver.load_compute_driver():


```
def load_compute_driver(virtapi, compute_driver=None):
    
    """
    Load a compute driver module.

    Load the compute driver module specified by the compute_driver
    configuration option or, if supplied, the driver name supplied as an
    argument.

    Compute drivers constructors take a VirtAPI object as their first object
    and this must be supplied.

    :param virtapi: a VirtAPI instance
    :param compute_driver: a compute driver name to override the config opt
    :returns: a ComputeDriver instance
    """
        if not compute_driver:
        compute_driver = CONF.compute_driver

    if not compute_driver:
        LOG.error(_LE("Compute driver option required, but not specified"))
        sys.exit(1)

    LOG.info(_LI("Loading compute driver '%s'"), compute_driver)
    try:
        driver = importutils.import_object_ns('nova.virt',
                                              compute_driver,
                                              virtapi)
        return utils.check_isinstance(driver, ComputeDriver)
    except ImportError:
        LOG.exception(_LE("Unable to load the virtualization driver"))
        sys.exit(1)
```

此处的compute_driver的值有nova.conf中的参数决定。至此，创建部署流程分析到了对应驱动中driver.py中的spwan函数。未完待续。本文初学openstack，如有错误，请批评指正。