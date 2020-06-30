---
title: "components"
linkTitle: "components"
type: docs
description: >
    Compose kustomizations.
---

Each entry in this list must be a path (or URL) referring to a Kustomize component  _directory_, e.g.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

components:
- ../../components/component-a
- github.com/kubernetes-sigs/kustomize/examples/components?ref=test-branch
```

Components will be read and processed in order.

[hashicorp URL]: https://github.com/hashicorp/go-getter#url-format

Directory specification can be relative, absolute, or part of a URL.  URL specifications should
follow the [hashicorp URL] format.  The directory must contain a `kustomization.yaml` file, and it _must_ have the following `apiVersion` and `kind` (which are different to normal Kustomization files):

```yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component
```

Component `kustomization.yaml` files are otherwise identical to 'normal' Kustomization's - the different `kind` is used to document the fact that they need to be processed as a component (described in the following section), and should be added to the `components` list instead of `bases`.

'Normal' Kustomizations are only aware of the resources that they (or their child `bases`, `components`, etc.) introduce.  They cannot affect resources introduced by their parent or sibling Kustomizations.  For exam ple, given `kust-a` as a parent Kustomization, `kust-b`, `kust-c` are normal Kustomizations, and `kust-d` is another Kustomization with `kust-e` defined in its resources:

`kust-a/kustomization.yaml`:
```
...
resources:
  - ../kust-b
  - ../kust-c
  - ../kust-d
...
```

We effectively have the following processing structure.

```
           +------+
           |kust-a|
           +------+
               |
    +----------+----------+
    |          |          |
+------+   +------+   +------+
|kust-b|   |kust-c|   |kust-d|
+------+   +------+   +------+
                          |
                      +------+
                      |kust-e|
                      +------+
``` 

The processing scope of an individual Kustomization is itself and its children.  For example, `kust-d` can only modify ConfigMaps defined by (its child) `kust-e`.

Components are different - their processing scope is the processing scope of their parent after all of their parent's resources and previous components have been processed, but before the other elements of the parent have been applied (for example namespace declarations and patches).  The effect is (approximately) as if the parent's resources have been moved to the start of the component's `resources` list.

For example, if we change `kust-d` to a component `comp-d` and add it to `kust-a`'s `components` list:

`kust-a/kustomization.yaml`:
```
...
resources:
  - ../kust-b
  - ../kust-c

components:
  - ../comp-d
...
```

The processing scope is now effectively:

```
           +------+
           |kust-a|
           +------+
               |
           +------+
           |comp-d|
           +------+
               |
    +----------+----------+
    |          |          |
+------+   +------+   +------+
|kust-b|   |kust-c|   |kust-e|
+------+   +------+   +------+
``` 

`comp-d` now also has `kust-b` and `kust-c` in scope, so for example could modify one of their ConfigMaps.