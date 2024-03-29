# Eclipse Mosquitto lightweight MQTT Message Broker

[Return to Application List](../../)

* Kustomize based ArgoCD application deployment
* Deployed as a Statefulset with a small 10mi Longhorn Persistent Storage Volume for data

Review files in `mosquitto/workloads/mosquitto/base/application` to populate the two ConfigMaps:

* `configmap-config.yaml` creates the `mosquitto.conf` configuration file
* `configmap-passwd.yaml` creates the encoded User & Password authentication file

Review `mosquitto/kustomization.yaml`

* Set the initial image version
* Service type is `LoadBalancer` and specific IP Address can be defined

```yaml
images:
  - name: eclipse-mosquitto
    newTag: 2.0.14

# Set Service to LoadBalancer and Specify IP Address to use
patches:
  - patch: |-
      - op: add
        path: /spec/type
        value: LoadBalancer
      - op: add
        path: /spec/loadBalancerIP
        value: 192.168.10.241
    target:
      kind: Service
```

Review `mosquitto/applications/mosquitto.yaml`

* Set `repoURL` source to the path of your dedicated ArgoCD repository

```yaml
  source:
    repoURL:  https://github.com/<USER_NAME>/<REPO_NAME>.git
    targetRevision: HEAD
    path: workloads/mosquitto
```

[Return to Application List](../../)
