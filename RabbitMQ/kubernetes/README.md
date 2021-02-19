# RabbitMQ for Kubernetes

### 准备基础镜像

### 创建命名空间Namespace.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: rabbitmq
```

### 创建ConfigMap.yaml
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: rabbitmq
  namespace: rabbitmq
  labels:
    app: rabbitmq
data:
  enabled_plugins: |
    [
      rabbitmq_shovel,
      rabbitmq_shovel_management,
      rabbitmq_federation,
      rabbitmq_federation_management,

      rabbitmq_consistent_hash_exchange,
      rabbitmq_management,
      rabbitmq_peer_discovery_k8s,

      rabbitmq_delayed_message_exchange,

      rabbitmq_prometheus
    ].

  rabbitmq.conf: |
    ## RabbitMQ configuration
    ## Ref: https://github.com/rabbitmq/rabbitmq-server/blob/master/docs/rabbitmq.conf.example

    ## Clustering
    cluster_formation.peer_discovery_backend  = rabbit_peer_discovery_k8s
    cluster_formation.k8s.host = kubernetes.rabbitmq.svc.cluster.local
    cluster_formation.k8s.address_type = hostname
    cluster_formation.node_cleanup.interval = 10
    
    # Set to false if automatic cleanup of absent nodes is desired.
    # This can be dangerous, see http://www.rabbitmq.com/cluster-formation.html#node-health-checks-and-cleanup.
    cluster_formation.node_cleanup.only_log_warning = true
    cluster_partition_handling = autoheal
    ## The default "guest" user is only permitted to access the server
    ## via a loopback interface (e.g. localhost)
    loopback_users.guest = false

    management.load_definitions = /etc/definitions/definitions.json

    ## Memory-based Flow Control threshold
    vm_memory_high_watermark.absolute = 256MB
```

### 创建Secret.yaml
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rabbitmq
  namespace: rabbitmq
  labels:
    app: rabbitmq
type: Opaque
data:
  rabbitmq-username: "Z3Vlc3Q="
  rabbitmq-password: "ekIwN1lYa0NQSzRvS0RTemNCOWxDMFBF"
  rabbitmq-management-username: "bWFuYWdlbWVudA=="
  rabbitmq-management-password: "dVJLV1pyZkI3SFdTQzdwVGpydFB2a1c5"
  rabbitmq-erlang-cookie: "bmZmbHdzajI4TUlxTzlqNElrZlI5cUVLeXp4czRNZ3U="
  definitions.json: "ewogICJnbG9iYWxfcGFyYW1ldGVycyI6IFsKICAgIAogIF0sCiAgInVzZXJzIjogWwogICAgewogICAgICAibmFtZSI6ICJtYW5hZ2VtZW50IiwKICAgICAgInBhc3N3b3JkIjogInVSS1dacmZCN0hXU0M3cFRqcnRQdmtXOSIsCiAgICAgICJ0YWdzIjogIm1hbmFnZW1lbnQiCiAgICB9LAogICAgewogICAgICAibmFtZSI6ICJndWVzdCIsCiAgICAgICJwYXNzd29yZCI6ICJ6QjA3WVhrQ1BLNG9LRFN6Y0I5bEMwUEUiLAogICAgICAidGFncyI6ICJhZG1pbmlzdHJhdG9yIgogICAgfQogIF0sCiAgInZob3N0cyI6IFsKICAgIHsKICAgICAgIm5hbWUiOiAiLyIKICAgIH0KICBdLAogICJwZXJtaXNzaW9ucyI6IFsKICAgIHsKICAgICAgInVzZXIiOiAiZ3Vlc3QiLAogICAgICAidmhvc3QiOiAiLyIsCiAgICAgICJjb25maWd1cmUiOiAiLioiLAogICAgICAicmVhZCI6ICIuKiIsCiAgICAgICJ3cml0ZSI6ICIuKiIKICAgIH0KICBdLAogICJwYXJhbWV0ZXJzIjogWwogICAgCiAgXSwKICAicG9saWNpZXMiOiBbCiAgICAKICBdLAogICJxdWV1ZXMiOiBbCiAgICAKICBdLAogICJleGNoYW5nZXMiOiBbCiAgICAKICBdLAogICJiaW5kaW5ncyI6IFsKICAgIAogIF0KfQ=="
```

### 创建Service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-discovery
  namespace: rabbitmq
  labels:
    app: rabbitmq-discovery
spec:
  clusterIP: None
  ports:
    - name: http
      protocol: TCP
      port: 15672
      targetPort: http
    - name: amqp
      protocol: TCP
      port: 5672
      targetPort: amqp
    - name: epmd
      protocol: TCP
      port: 4369
      targetPort: epmd
  publishNotReadyAddresses: true
  selector:
    app: rabbitmq

---

apiVersion: v1
kind: Service
metadata:
  name: rabbitmq-svc
  namespace: rabbitmq
  labels:
    app: rabbitmq-svc
spec:
  ports:
    - name: http
      protocol: TCP
      port: 15672
      targetPort: http
    - name: amqp
      protocol: TCP
      port: 5672
      targetPort: amqp
    - name: epmd
      protocol: TCP
      port: 4369
      targetPort: epmd
    - name: metrics
      protocol: TCP
      port: 15692
      targetPort: metrics
  selector:
    app: rabbitmq
```

### 创建有状态应用StatefulSet.yaml
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: rabbitmq
  namespace: rabbitmq
  labels:
    app: rabbitmq
spec:
  serviceName: rabbitmq-discovery
  replicas: 3
  selector:
    matchLabels:
      app: rabbitmq
  template:
    metadata:
      labels:
        app: rabbitmq
    spec:
      securityContext:
        fsGroup: 101
        runAsGroup: 101
        runAsNonRoot: true
        runAsUser: 100
      initContainers:
        - name: bootstrap
          image: busybox:1.30.1
          imagePullPolicy: IfNotPresent
          command: ['sh']
          args:
            - "-c"
            - |
              set -ex
              cp /configmap/* /etc/rabbitmq
              echo "${RABBITMQ_ERLANG_COOKIE}" > /var/lib/rabbitmq/.erlang.cookie
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: RABBITMQ_MNESIA_DIR
              value: /var/lib/rabbitmq/mnesia/rabbit@$(POD_NAME).rabbitmq-discovery.rabbitmq.svc.cluster.local
            - name: RABBITMQ_ERLANG_COOKIE
              valueFrom:
                secretKeyRef:
                  name: rabbitmq
                  key: rabbitmq-erlang-cookie
          volumeMounts:
            - name: configmap
              mountPath: /configmap
            - name: config
              mountPath: /etc/rabbitmq
            - name: data
              mountPath: /var/lib/rabbitmq
      containers:
        - name: rabbitmq
          image: registry.cn-hangzhou.aliyuncs.com/zpparts2/rabbitmq:3.8.0-alpine
          imagePullPolicy: IfNotPresent
          ports:
            - name: epmd
              protocol: TCP
              containerPort: 4369
            - name: amqp
              protocol: TCP
              containerPort: 5672
            - name: http
              protocol: TCP
              containerPort: 15672
            - name: metrics
              protocol: TCP
              containerPort: 15692
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - 'wget -O - -q --header "Authorization: Basic `echo -n \"$RABBIT_MANAGEMENT_USER:$RABBIT_MANAGEMENT_PASSWORD\"
                | base64`" http://localhost:15672/api/healthchecks/node | grep -qF "{\"status\":\"ok\"}"'
            failureThreshold: 6
            initialDelaySeconds: 120
            periodSeconds: 10
            timeoutSeconds: 5
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - 'wget -O - -q --header "Authorization: Basic `echo -n \"$RABBIT_MANAGEMENT_USER:$RABBIT_MANAGEMENT_PASSWORD\"
                | base64`" http://localhost:15672/api/healthchecks/node | grep -qF "{\"status\":\"ok\"}"'
            failureThreshold: 6
            initialDelaySeconds: 20
            periodSeconds: 5
            timeoutSeconds: 3
          env:
            - name: MY_POD_NAME
              valueFrom:
                fieldRef:
                  apiVersion: v1
                  fieldPath: metadata.name
            - name: RABBITMQ_USE_LONGNAME
              value: "true"
            - name: RABBITMQ_NODENAME
              value: rabbit@$(MY_POD_NAME).rabbitmq-discovery.rabbitmq.svc.cluster.local
            - name: K8S_HOSTNAME_SUFFIX
              value: .rabbitmq-discovery.rabbitmq.svc.cluster.local
            - name: K8S_SERVICE_NAME
              value: rabbitmq-discovery
            - name: RABBITMQ_ERLANG_COOKIE
              valueFrom:
                secretKeyRef:
                  name: rabbitmq
                  key: rabbitmq-erlang-cookie
            - name: RABBIT_MANAGEMENT_USER
              valueFrom:
                secretKeyRef:
                  name: rabbitmq
                  key: rabbitmq-management-username
            - name: RABBIT_MANAGEMENT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: rabbitmq
                  key: rabbitmq-management-password
          resources:
            limits:
              cpu: 3
              memory: 500Mi
            requests:
              cpu: 200m
              memory: 500Mi
          volumeMounts:
            - name: data
              mountPath: /var/lib/rabbitmq
            - name: config
              mountPath: /etc/rabbitmq
            - name: definitions
              mountPath: /etc/definitions
              readOnly: true
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 1
              podAffinityTerm:
                topologyKey: "kubernetes.io/hostname"
                labelSelector:
                  matchLabels:
                    app: rabbitmq
      volumes:
        - name: config
          emptyDir: {}
        - name: configmap
          configMap:
            name: rabbitmq
        - name: definitions
          secret:
            secretName: rabbitmq
            items:
              - key: definitions.json
                path: definitions.json
  volumeClaimTemplates:
    - metadata:
        name: data
        annotations:
          volume.beta.kubernetes.io/storage-class: "nfs"
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 20Gi
```