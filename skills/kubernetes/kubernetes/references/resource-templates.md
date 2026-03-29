# Kubernetes Resource Templates

## Production-Ready Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: APP_NAME
  labels:
    app.kubernetes.io/name: APP_NAME
    app.kubernetes.io/instance: APP_NAME-ENVIRONMENT
    app.kubernetes.io/version: "VERSION"
    app.kubernetes.io/component: COMPONENT
    app.kubernetes.io/part-of: PLATFORM
    app.kubernetes.io/managed-by: kustomize
spec:
  replicas: 2
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app.kubernetes.io/name: APP_NAME
      app.kubernetes.io/instance: APP_NAME-ENVIRONMENT
  template:
    metadata:
      labels:
        app.kubernetes.io/name: APP_NAME
        app.kubernetes.io/instance: APP_NAME-ENVIRONMENT
        app.kubernetes.io/version: "VERSION"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: APP_NAME
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 30
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: APP_NAME
      containers:
        - name: APP_NAME
          image: REGISTRY/APP_NAME:VERSION
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: APP_ENV
              value: "ENVIRONMENT"
          envFrom:
            - configMapRef:
                name: APP_NAME-config
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 15
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /healthz
              port: http
            failureThreshold: 30
            periodSeconds: 10
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
```

## StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: APP_NAME
spec:
  serviceName: APP_NAME
  replicas: 3
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: APP_NAME
  template:
    metadata:
      labels:
        app.kubernetes.io/name: APP_NAME
    spec:
      # ... same security context, probes, resources as Deployment
      containers:
        - name: APP_NAME
          image: REGISTRY/APP_NAME:VERSION
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: oci-bv           # Adjust for your cloud
        resources:
          requests:
            storage: 10Gi
```

## CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: APP_NAME-job
spec:
  schedule: "0 2 * * *"          # Daily at 2am
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 3600
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: APP_NAME
          securityContext:
            runAsNonRoot: true
            runAsUser: 1000
          containers:
            - name: job
              image: REGISTRY/APP_NAME:VERSION
              command: ["/app/job"]
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "1000m"
                  memory: "512Mi"
              securityContext:
                readOnlyRootFilesystem: true
                allowPrivilegeEscalation: false
                capabilities:
                  drop: [ALL]
```

## HorizontalPodAutoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: APP_NAME
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: APP_NAME
  minReplicas: 2
  maxReplicas: 10
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 25
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 30
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

## ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: APP_NAME
  labels:
    app.kubernetes.io/name: APP_NAME
automountServiceAccountToken: false
```

## ExternalSecret (ESO)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: APP_NAME
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend       # or oci-vault, aws-secretsmanager
    kind: ClusterSecretStore
  target:
    name: APP_NAME-secrets
    creationPolicy: Owner
  data:
    - secretKey: db-password
      remoteRef:
        key: /app/database/password
    - secretKey: api-key
      remoteRef:
        key: /app/api-key
```

## Ingress (NGINX)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: APP_NAME
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app.example.com
      secretName: app-tls
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: APP_NAME
                port:
                  name: http
```
