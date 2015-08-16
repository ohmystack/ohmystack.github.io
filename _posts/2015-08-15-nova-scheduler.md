---
layout: post
title: Nova Scheduler
description: "主要介绍 filter_scheduler"
category: articles
tags: [OpenStack, Nova]
---

```python
get_filtered_hosts
# Used at `nova/scheduler/filter_scheduler.py`
```

```python
HostManager.filter_handler.get_filtered_objects(filter_classes, hosts, filter_properties)
# Used at `HostManager.get_filtered_hosts`
```
这里的 hosts 是一个 `HostState` 的 list

默认通常我们用下面这些 filters

```python
ipdb> pp CONF.scheduler_default_filters
['RetryFilter',
'AvailabilityZoneFilter',
'RamFilter',
'CoreFilter',
'ComputeFilter',
'ComputeCapabilitiesFilter',
'ImagePropertiesFilter',
'NumInstancesFilter']
```

### 何为 filter？

基类在 `nova/filters.py`

`BaseFilterHandler` 的 `get_filtered_objects` 就是让 xxxFilter 执行 `filter_all(objs, filter_properties)`，
而 `filter_all` 里面是循环执行 `_filter_one`。

整个 `FilterHandler` 返回的结果再被下一个 filter 过滤。

nova-scheduler (`nova/scheduler/filters/__init__.py`) 有 `BaseHostFilter` 和 `HostFilterHandler`。

`HostFilterHandler` 里面的 `host_passes(self, host_state, filter_properties)` 方法，如果 filter 通过，则返回 `True`，这个 host 就被允许进入下一轮过滤。

#### **RetryFilter**

如果允许 retry，那么在传进来的 filter_properties 里，会有 'retry' 字段。
类似 'retry': {'hosts': [], 'num_attempts': 1},

#### **AvailabilityZoneFilter**

如果 `filter_properties` 里对 `availability_zone` 有要求。
则与 host 的 `aggregate_metadata` 里 `availability_zone` 进行比较。
如果没有，则与 `CONF.default_availability_zone` 进行比较。

#### **CoreFilter**

根据 host 上还可用的 vcpu 数量进行过滤。

#### **ComputeFilter**

`service_is_up` 且没有 `host_state.service['disabled']`

#### **ComputeCapabilitiesFilter**

检查 `instance_type`

#### **ImagePropertiesFilter**

根据 HostState 里的 capabilities，与 `filter_properties` 里 `image.properties` 的 `request_spec.image.properties` 进行过滤。

```python
image_props.get('architecture', None)
image_props.get('hypervisor_type', None)
image_props.get('vm_mode', None)
```

#### **NumInstancesFilter**

检查 `host_state.num_instances < CONF.max_instances_per_host`

其实这几个自带的 Filter 都很容易，不用在这里多介绍，直接去看源码就好了。
