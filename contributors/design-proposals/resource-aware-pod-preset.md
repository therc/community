# Resource-aware Pod Preset

  * [Abstract](#abstract)
  * [Motivation](#motivation)
  * [Use Cases](#use-cases)
  * [Proposed Changes](#proposed-changes)
    * [API objects](#api-objects)
  * [Alternatives](#alternatives)
  * [Examples](#examples)
    * [Tensorflow](#tensorflow)
    
## Abstract

This document proposes a policy object that lets Pods which use specific runtime resources to abstract
away some of their details and defer them until creation time. For example, a Pod using GPU resources
might need to access low-level libraries that are inherently host-specific (sometimes kernel-specific,
even). Currently, such host volume mounts are not very portable, as different hosts might run
different Linux distributions. The problem is thus better tackled at the cluster and namespace level.

## Motivation

This proposal closely tracks and aligns with the
[Pod Preset proposal](https://github.com/therc/community/blob/master/contributors/design-proposals/pod-preset.md).
The motivations are virtually identical. The only nuance is the matching being performed based on resources,
not labels. Refer to that document for a fuller picture.

## Use Cases

Just looking at Nvidia hardware alone:

 - As an user, I want to run a plain Tensorflow image with a simple YAML file, without having to
 hardcode or worry about the vagaries of where on the host Nvidia's low-level libraries reside.
 - As an app developer, I, too, want to spend my time writing CUDA code, not dealing with low-level
 details. I would like to run the `nvidia-smi` monitoring tool in my container without shipping it
 myself (for licensing and compatibility issues) or hardcoding its location.
 - As a cluster admin, I want the flexibility to migrate clusters to new base images where low-level
 libraries and binaries might live in different directories, without forcing my users to think about
 the issue or, worse, create different templates for different clusters.
 - As a cluster admin, I want to increase the `CUDA_CACHE_MAXSIZE` variable from the default because
 our existing Docker images come with pre-built code for older cards, but for the newer cards the JIT
 compiler gets called at runtime and its output is never cached.
 - As a cluster admin, I want to expose a host directory with low-level GPU debugging tools only to
 pods in the `dev` namespace.

For other resources:
 - As an admin, I want all pods using the opaque integer resource `localssd` to automatically mount
 a host directory of my choice, presumably where my SSD is mounted.
 - As an user, I just want to use the fast, local SSD disk without having to know where it lives.

## Proposed Changes

### API objects

Two basic options are available: augment `PodPresetSpec` or create a parallel
`ResourceAwarePodPreset`/`ResourceAwarePodPresetSpec` combination.

`ProdPresetSpec` currently lives in the `settings` group and looks like this:

```go
type PodPresetSpec struct {
    // Selector is a label query over a set of resources, in this case pods.
    // Required.
    Selector      unversioned.LabelSelector
    // Env defines the collection of EnvVar to inject into containers.
    // +optional
    Env           []EnvVar
    // EnvFrom defines the collection of EnvFromSource to inject into
    // containers.
    // +optional
    EnvFrom       []EnvFromSource
    // Volumes defines the collection of Volume to inject into the pod.
    // +optional
    Volumes       []Volume `json:omitempty`
    // VolumeMounts defines the collection of VolumeMount to inject into
    // containers.
    // +optional
    VolumeMounts  []VolumeMount
}
```

Ideally, the simplest change would be to add a new kind of selector, perhaps named `ResourceSelector`.
This is simpler both in terms of implementation and mental model for users: one object and one concept,
rather than two, similar ones. The LabelSelector is presently required, but it might have to become
optional, as long as a ResourceSelector is present. TBD: does this cause migration issues? If both
selectors are present, they would be `AND`ed together. If an admin wanted to `OR` them, they would
create two separate policies.

Alternatively, an entirely new object (and controller) could be created. This would require parallel
API objects and their associated logic to be maintained.

A `ResourceSelector` could be a simple list of resource names:

```go
ResourceSelector   []ResourceName
```

It's not clear if there's any value in more complex selectors, such as
`alpha.kubernetes.io/nvidia-gpu > 2` or `not alpha.kubernetes.io/nvidia-gpu`.

## Alternatives

An obvious alternative would use a plain `PodPreset` and forcing users to add
a new label (`use-gpu`?) on top of the existing GPU resource requests/limits.
Such an approach works, but is obviously error-prone and will lead to a
proliferation across the Kubernetes user base of similar, but not identical labels.

## Examples

### Tensorflow

**Pod Preset:**

```yaml
kind: PodPreset
apiVersion: settings/v1alpha1
metadata:
  name: gpu-setup
  namespace: myns
spec:
  resourceSelector:
    - alpha.kubernetes.io/nvidia-gpu
  volumeMounts:
    - name: binaries
      mountPath: /opt/bin
      readOnly: true
    - name: libraries
      mountPath: /usr/local/lib/nvidia
      readOnly: true
  volumes:
    - name: binaries
      hostPath:
        path: /opt/bin
    - name: libraries
      hostPath:
        path: /opt/lib64
```


