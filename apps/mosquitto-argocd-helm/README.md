# Eclipse Mosquitto lightweight MQTT Message Broker

[Return to Application List](../../)

* Helm based ArgoCD application deployment
 Based on [Bernd Schorgers](https://bjw-s.github.io/helm-charts/docs/) standard `app-template` which replaces the deprecated [k8s@home](https://github.com/k8s-at-home/charts) common template
* Deployed as a Statefulset with a 100Mi Persistent Storage Volume for configuration

Review file `mosquitto-argocd-helm/applications/mosquitto.yaml`

* Define the ArgoCD project to assign this application to
* ArgoCD uses `default` project by default

  ```yaml
  spec:
    project: default
  ```

* The source repository URL specifies the version of the `app-template` as well as the path within your GitHub repository for location of the `values.yaml`:

  ```yaml
    - repoURL: https://bjw-s.github.io/helm-charts
      chart: app-template
      targetRevision: 1.5.0
      helm:
        valueFiles:
          - $values/workloads/mosquitto/values.yaml
  ```

* The `$values` variable defines your GitHub repository. Set your `USER_NAME` and `REPO_NAME`.

  ```yaml
    - repoURL: 'https://github.com/<USER_NAME>/<REPO_NAME>.git'
      targetRevision: HEAD
      ref: values
  ```

* Define the namespace to deploy to, this defaults to `mosquitto`.

  ```yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: mosquitto
  ```

---

Review file `mosquitto-argocd-helm/workloads/mosquitto/values.yaml`

* An `initContainer` named `01-init-config` is used to copy the Mosquitto password file from `/data/mosquitto_secret/` to `/data/external_config/` and then chmod the password file to `440`.
  * This was required to get around Kubernetes mounted config files being root owned and world readable.
  * Mosquitto will display a warning message if the password file is NOT owned by `mosquitto` account or if the file is world readable. The warning messages states that a future version of Mosquitto will refuse to start.  This init container corrects these conditions and the warning message will not be logged.

* Service type is `LoadBalancer` and specific IP Address can be defined:

  ```yaml
    service:
      main:
        enabled: true
        annotations:
          kube-vip.io/loadbalancerIPs: 192.168.10.222
        ports:
          http:
            enabled: false
          mqtt:
            enabled: true
            port: 1883
        type: LoadBalancer
        loadBalancerIP: 192.168.10.222
  ```

* The `podSecurityContext` sets the runAs User and Group to `1883` which is the user ID and Group ID of the `mosquitto` user defined within the container.
  * This helps to insure that the files copied within the `initContainer` are correctly owned to prevent the Mosquitto startup warning message described above.

Define each username and password needed to authenticate with the broker:

* Mosquitto uses `htpasswd` to only store a hash of the password, not the actual password
  * Password hashes are stored in a `configMap`, a Kubernetes `secret` is not required
* Add one user per line as `userid`:`hashed password`

  ```yaml
  configMaps:
    ...

    mosquitto-passwords:
      enabled: true
      data:
        mosquitto_pwd: |
          mqtt-user:$6$ZLO-[EXAMPLE]-Qeow==
  ```

* Update Mosquitto Configuration file

  ```yaml
  configMaps:
    ...
  
    mosquitto-configmap:
      enabled: true
      data:
        mosquitto.conf: |
          per_listener_settings false
          listener 1883
          allow_anonymous false
          persistence true
          persistence_location /data
          connection_messages false
          autosave_interval 120
          password_file /mosquitto/external_config/mosquitto_pwd
  ```

  * The default values are reasonable, quick summary:
    * `allow_anonymous` when `false` requires all connections to be authenticated (have a ID and password)
    * `persistence` when `true` will save the in-memory database to path specified by `persistence_location` every `autosave_interval` seconds

* Define Persistent Storage for MQTT Data

  ```yaml
  persistence:
    data:
      enabled: true
      mountPath: /data
      type: pvc
      accessMode: ReadWriteOnce
      size: 100Mi
      retain: true
      # storageClass:
      # existingClaim:
      # volumeName:
  ```

[Return to Application List](../../)
