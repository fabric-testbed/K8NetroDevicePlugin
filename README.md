# linux-vfio-k8s-dpi

## Introduction

This software is a kubernetes [device plugin](https://kubernetes.io/docs/concepts/cluster-administration/device-plugins/) that exposes PCI devices from the underlying system and makes them available as resources for the kubernetes node.

## Building

### Plain binary

```bash
cd cmd/vfio
go build .
```

### Docker

```bash
docker build -t vfio:latest .
```

## Running

NOTE: This process is not finalized yet.

## As a DaemonSet

```
# Deploy the device plugin
kubectl apply -f manifests/vfio-ds.yml

# Optionally you can now test it using an example consumer
kubectl apply -f examples/vfio-consumer.yml
kubectl exec -it vfio-consumer -- ls /dev/vfio
```

## API

Node description:

```yaml
Capacity:
 ...
 devices.netronome.io/19ee_4000:  3
 devices.netronome.io/19ee_4000:  1
 devices.netronome.io/19ee_4000:  1
 devices.netronome.io/19ee_4000:  2
 devices.netronome.io/19ee_4000:  1
 ...
```

Devices are reported as scalar resources, identified by vendor ID and device ID. The format shown in node description is `namespace/vendorID_product_ID: #ofAvailableDevices)`.

Pod:

```yaml
spec:
  containers:
  - name: demo
    ...
    resources:
      requests:
              devices.netronome.io/19ee_4000: 1
      limits:
              devices.netronome.io/19ee_4000: 1
```

Pod device assignment is derived from device plugin requirements. Snippet above would make sure that

* the pod is scheduled onto node with Tesla P4 installed,
* the device is bound to vfio-pci,
* pod sees VFIO path (/dev/vfio/*).

## Issues

### IOMMU

Currently, IOMMU groups are not enforced at the plugin's level in reporting capabilities. When allocating a device from group that contains other devices, all devices are unbound from their respective drivers but not assigned or reported as unhealthy. This has an unfortunate consequence, best demonstrated by example:

Let `1` be IOMMU group containing devices `A`, `B` and `C`.

If container `a` requests device `A`,

* `A`, `B` and `C` are unbound from their original driver,
* `A`, `B` and `C` are bound to vfio-pci driver,
* `a` gets it's host/container path of the VFIO endpoint (/dev/vfio/1 in this case).

If then container `b` requests device `B`,

* the device is already bound to vfio-pci (which is not a problem),
* `b` is assigned host/container path to the VFIO endpoint (which, same as with `a`, is /dev/vfio/1).

If we start a VM `α` in container `a` with device `A` assigned, and VM `β` in container `b` with device `B` assigned, only one of these VMs would start successfully: we've violated IOMMU restrictions.

The scenario can be solved in several ways -

* assign the whole group to the container,
* report devices `B`, `C` as unhealthy when `a` is created, or devices `A`, `C` when `b` is created.
* ?

The first approach is not feasible as IOMMU groups could be different on another node, e.g. `1` would contain `A` whereas `B` and `C` might be in different group(s).

The second approach has an issue on it's own: when `a` or `b` is destroyed, the plugin is not informed of such event. Adding that capability to the device plugin API may not be desired as it implies a stateful nature of the device. Depending on other device plugin's state management needs, it could turn out to be the right solution.
