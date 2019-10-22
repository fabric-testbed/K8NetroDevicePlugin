# linux-netronome-vfio-k8s-dpi

## Introduction

This software is a kubernetes [device plugin](https://kubernetes.io/docs/concepts/cluster-administration/device-plugins/) that exposes PCI devices from the underlying system and makes them available as resources for the kubernetes node.

This Device Plugin is based on Kubevirt Device Plugin Manager mentioned on Kubernetees.io. See Device Plugin Manager documentaion on https://godoc.org/github.com/kubevirt/device-plugin-manager/pkg/dpm

## Building
```
make build
```

## Device Plugin Details
Device Plugin details available [here](./cmd/vfio/README.md)
