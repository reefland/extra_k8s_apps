# External Snapshotter

The CSI snapshotter is part of Kubernetes implementation of [Container Storage Interface (CSI)](https://github.com/container-storage-interface/spec).

Project Documentation: <https://github.com/kubernetes-csi/external-snapshotter>

This can be used to enable snapshot support for CSI like:

* [Democratic-CSI](https://github.com/reefland/ansible-k3s-argocd-renovate/blob/master/docs/democratic-csi-settings.md)
* [Rook-Ceph](https://github.com/reefland/extra_k8s_apps/tree/master/rook-ceph-argocd-helm) - See [Project Documentation](https://rook.io/docs/rook/v1.10/Storage-Configuration/Ceph-CSI/ceph-csi-snapshot/) on Snapshot support

[Return to Application List](../)

* Kustomize based ArgoCD application deployment
* Deployed as a Statefulset

---

## Installation Configuration

Update file in `external-snapshotter-argocd-kustomize/workloads/gitea/kustomization.yaml` to define specifics about your Kubernetes environment:

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

[Return to Application List](../)
