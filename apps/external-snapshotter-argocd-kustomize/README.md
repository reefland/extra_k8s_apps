# External Snapshotter

The CSI snapshotter is part of Kubernetes implementation of [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec).

Project Documentation: <https://github.com/kubernetes-csi/external-snapshotter>

This can be used to enable snapshot support for CSI like:

* [Democratic-CSI](https://github.com/reefland/ansible-k3s-argocd-renovate/blob/master/docs/democratic-csi-settings.md)
* [Rook-Ceph](../rook-ceph-argocd-helm/) - See [Project Documentation](https://rook.io/docs/rook/v1.10/Storage-Configuration/Ceph-CSI/ceph-csi-snapshot/) on Snapshot support

[Return to Application List](../../)

* Kustomize based ArgoCD application deployment
* Deployed as a Statefulset

---

## Installation Configuration

Update file in `external-snapshotter-argocd-kustomize/workloads/gitea/kustomization.yaml` to define specifics about your Kubernetes environment:

* Define the initial versions of the CSI Provisioner, Snapshotter, Snapshot Controller and hostpath plugin:

  ```yaml
  images:
    # https://github.com/kubernetes-csi/external-provisioner/releases
    - name: registry.k8s.io/sig-storage/csi-provisioner
      newTag: v3.4.0
    # https://github.com/kubernetes-csi/external-snapshotter/releases
    - name: registry.k8s.io/sig-storage/snapshot-controller
      newTag: v6.2.1
    - name: registry.k8s.io/sig-storage/csi-snapshotter
      newTag: v6.2.1
    # https://github.com/kubernetes-csi/csi-driver-host-path/releases
    - name: registry.k8s.io/sig-storage/hostpathplugin
      newTag: v1.11.0
  ```

* Define Revision History Limits, belows sets to `3`:

  ```yaml
  patches:
    # Set revisionHistoryLimit on StatefulSet
    - patch: |-
        - op: add
          path: /spec/revisionHistoryLimit
          value: 3
      target:
        kind: StatefulSet
        name: csi-snapshotter
  ```

---

[Return to Application List](../../)
