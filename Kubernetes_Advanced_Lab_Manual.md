# Kubernetes Advanced Lab — Step-by-step Hands-on Manual

Kubernetes Advanced Lab — Step-by-step Hands-on Manual
(For: 45-minute advanced lab)

Purpose:
This manual is a single, copy-paste friendly guide you can use on Windows (PowerShell).
It includes: explanation → manifest → exact PowerShell commands to run (no steps skipped).

Environment assumptions:
- Windows machine with Docker installed and running.
- PowerShell run AS ADMINISTRATOR for hosts file edits.
- minikube (recommended) installed. This guide uses minikube with Docker driver.
- kubectl installed and available in PATH.

Paths used in this guide:
Base folder (your workspace):
F:\Symbiosis DevOps 2025-26\3rd Lecture\k8s advance
All Kubernetes manifest files live in the subfolder: manifests

You will find both the YAML manifests and exact PowerShell here-strings used to create them.
0) Cluster Setup

Start minikube and enable required addons (ingress and metrics-server for HPA).
Run these in PowerShell (Admin).

PowerShell / Manifest (copy-paste):

minikube start --driver=docker

# enable ingress and metrics-server
minikube addons enable ingress
minikube addons enable metrics-server

# verify
kubectl cluster-info
minikube ip


--------------------------------------------------------------------------------

1) Create folders

You told me your base folder. We'll create the exact folder structure and a 'manifests' directory.
Run this in PowerShell (Admin).

PowerShell / Manifest (copy-paste):

$BASE = 'F:\Symbiosis DevOps 2025-26\3rd Lecture\k8s advance'
# create base and manifests folder
New-Item -ItemType Directory -Path $BASE -Force | Out-Null
New-Item -ItemType Directory -Path (Join-Path $BASE 'manifests') -Force | Out-Null
cd $BASE


--------------------------------------------------------------------------------

2) Create manifest files (PowerShell here-strings)

We create five manifest files inside the manifests folder. Copy-paste the PowerShell here-strings below (run in PowerShell Admin) — they create the YAML files exactly as used in the guide.
Files created:
- app.yaml (Deployment + NodePort Service)
- pvc-demo.yaml (PVC + Pod to demonstrate persistence)
- config-secret.yaml (ConfigMap + Secret + tester Pod)
- statefulset.yaml (headless Service + StatefulSet)
- ingress.yaml (Ingress resource pointing demo.local → demo-svc)


PowerShell / Manifest (copy-paste):

@'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
        - name: web
          image: nginx:1.23
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: demo-svc
spec:
  type: NodePort
  selector:
    app: demo
  ports:
    - port: 80
      targetPort: 80
'@ | Out-File -FilePath (Join-Path $BASE 'manifests\app.yaml') -Encoding utf8


@'
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
  - name: busy
    image: busybox
    command: ["sh","-c","echo hello > /data/hello.txt; sleep 3600"]
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: demo-pvc
'@ | Out-File -FilePath (Join-Path $BASE 'manifests\pvc-demo.yaml') -Encoding utf8


@'
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  APP_ENV: "production"
  WELCOME_MSG: "Hello from ConfigMap"

---
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASS: s3cr3t

---
apiVersion: v1
kind: Pod
metadata:
  name: config-secret-demo
spec:
  containers:
    - name: tester
      image: busybox
      command: ["sh","-c","echo Config APP_ENV=$APP_ENV; echo Secret user from secret: $DB_USER; cat /etc/config/WELCOME_MSG; sleep 3600"]
      env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: demo-config
              key: APP_ENV
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: demo-secret
              key: DB_USER
      volumeMounts:
        - name: cfg
          mountPath: /etc/config
        - name: sec
          mountPath: /etc/secret
          readOnly: true
  volumes:
    - name: cfg
      configMap:
        name: demo-config
    - name: sec
      secret:
        secretName: demo-secret
'@ | Out-File -FilePath (Join-Path $BASE 'manifests\config-secret.yaml') -Encoding utf8


@'
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "web"
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.23
          ports:
            - containerPort: 80
  volumeClaimTemplates:
    - metadata:
        name: www
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 50Mi
'@ | Out-File -FilePath (Join-Path $BASE 'manifests\statefulset.yaml') -Encoding utf8


@'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: demo.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: demo-svc
                port:
                  number: 80
'@ | Out-File -FilePath (Join-Path $BASE 'manifests\ingress.yaml') -Encoding utf8


--------------------------------------------------------------------------------

3) Apply base app

Apply base Deployment + Service, then test with port-forward.

PowerShell / Manifest (copy-paste):

cd $BASE
kubectl apply -f .\manifests\app.yaml
kubectl get deploy,pods,svc -o wide

# Quick test via port-forward (in same or new PowerShell window):
kubectl port-forward svc/demo-svc 8080:80
# Open http://localhost:8080 in browser or run: curl http://localhost:8080


--------------------------------------------------------------------------------

4) Rolling update & rollback

Commands to perform a rolling update and rollback. Watch pods with 'kubectl get pods -w' while updating.

PowerShell / Manifest (copy-paste):

# Trigger rolling update (change image tag)
kubectl set image deployment/demo-deploy web=nginx:1.25 --record
kubectl rollout status deployment/demo-deploy
kubectl rollout history deployment/demo-deploy

# Rollback to previous revision
kubectl rollout undo deployment/demo-deploy


--------------------------------------------------------------------------------

5) Scaling & HPA

Manual scaling and quick HPA commands (requires metrics-server).

PowerShell / Manifest (copy-paste):

# Manual scale
kubectl scale deployment/demo-deploy --replicas=4
kubectl get pods -l app=demo

# Quick HPA (auto create)
kubectl autoscale deployment demo-deploy --cpu-percent=50 --min=2 --max=5
kubectl get hpa -w

# Generate load (in another window) to trigger scaling
kubectl run -it --rm loadgen --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://demo-svc.default.svc.cluster.local; done"


--------------------------------------------------------------------------------

6) Persistent vs Ephemeral storage

Apply PVC + Pod example and verify file persistence.

PowerShell / Manifest (copy-paste):

kubectl apply -f .\manifests\pvc-demo.yaml
kubectl get pvc,pod pvc-demo
kubectl exec -it pvc-demo -- cat /data/hello.txt


--------------------------------------------------------------------------------

7) ConfigMaps & Secrets

Apply ConfigMap and Secret and verify pod logs (tester pod prints config and secret values).

PowerShell / Manifest (copy-paste):

kubectl apply -f .\manifests\config-secret.yaml
kubectl get configmap demo-config
kubectl get secret demo-secret
kubectl logs pod/config-secret-demo


--------------------------------------------------------------------------------

8) StatefulSet & Headless service

Apply headless service + StatefulSet. Observe stable pod names and PVCs.

PowerShell / Manifest (copy-paste):

kubectl apply -f .\manifests\statefulset.yaml
kubectl get sts,pods,pvc

# Delete a pod to show stable identity
kubectl delete pod web-1
kubectl get pods -w


--------------------------------------------------------------------------------

9) Ingress

Apply ingress and add demo.local to your hosts file so you can curl demo.local from Windows.

PowerShell / Manifest (copy-paste):

kubectl apply -f .\manifests\ingress.yaml
kubectl get ingress

# On Windows (PowerShell Admin) append hosts entry (replace IP as returned by minikube ip):
$minikubeIP = minikube ip
$hostsPath = 'C:\Windows\System32\drivers\etc\hosts'
"$minikubeIP`t demo.local" | Out-File -FilePath $hostsPath -Encoding ascii -Append
Get-Content $hostsPath

# Test
curl http://demo.local


--------------------------------------------------------------------------------

10) Cleanup

Commands to delete resources and stop/delete minikube at end of class.

PowerShell / Manifest (copy-paste):

kubectl delete -f .\manifests\ingress.yaml
kubectl delete -f .\manifests\statefulset.yaml
kubectl delete -f .\manifests\config-secret.yaml
kubectl delete -f .\manifests\pvc-demo.yaml
kubectl delete -f .\manifests\app.yaml

minikube stop
minikube delete


--------------------------------------------------------------------------------

Troubleshooting quick commands:
- Describe failing pod: kubectl describe pod <pod-name>
- View logs: kubectl logs <pod-name>
- PVC pending: kubectl describe pvc <pvc-name>
- HPA metrics: kubectl top pods (ensure metrics-server running)
- Ingress not working: ensure minikube addon ingress enabled and hosts entry exists

Security note: Kubernetes Secrets are base64-encoded by default. Do not store production secrets in plain manifests. Use external secret stores or enable encryption at rest.
