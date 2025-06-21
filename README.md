# my-web-app helm chart
This documents explains setting up a kubernetes cluster at Google cloud & helm chart creations for deploying two applications (Frontend & Backend) also shows the deployement and release management for those apps 
### Prerequisites
Google Cloud Platform (GCP) account
Creation of GKE clusters
The gcloud command-line tool (Cloud shell)
The kubectl command-line
The helm (version 3) 

### Part -1 Environment setup
- GCloud command to create the cluster with Cluster Name: helm-test-cluster
Region: us-central1
Number of Nodes: 2

```sh
$ gcloud container clusters create helm-test-cluster \
  --zone us-central1-a \
  --num-nodes 2 \
  --machine-type e2-medium
```
- Verification command of  kubectl was correctly pointing to a new GKE cluster  
```sh
$ gcloud container clusters get-credentials helm-test-cluster --zone us-central1-a 
```
 The above command used for authenticating to the kubernetes cluster and it 
 created kubeconfig file at user home directory at the cloudshell 
 the config file points to the cluster created with name "helm-test-cluster"
 
 - Kubectl command to verify the connections to the kubrnetes cluster
 and its showed the name of the cluster created above
```sh
$ kubectl config current-context
 gke_neat-striker-462705-a9_us-central1-a_helm-test-cluster
```
### Part 2 Helm Chart Creation

-  Directory structure of completed Helm chart
```sh
my-web-app/
├── charts/
└── templates/
    ├── tests/
    │   └── test-connection.yaml
    ├── _helpers.tpl
    ├── deployment.yaml
    ├── NOTES.txt
    ├── service.yaml
    ├── .helmignore
    ├── Chart.yaml
    └── values.yaml
```
- Contents of values.yaml file

```sh
components:
  frontend:
    # This will set the replicaset count
    replicaCount: 1
    # This sets the container image
    image:
      repository: nginx
      pullPolicy: IfNotPresent
      tag: "1.21.4"
    # This is to override the default name
    nameOverride: "ui"
    # This is for setting Kubernetes Labels to the Pod
    podLabels: {}
    # For Setting up service
    service:
      type: ClusterIP
      port: 80
    # For setting resource requirements
    resources: {}
    # Liveness and Readness probes for the pods
    livenessProbe:
      httpGet:
        path: /
        port: http
      initialDelaySeconds: 10
      periodSeconds: 10
      timeoutSeconds: 2        
      failureThreshold: 3         
      successThreshold: 1
    readinessProbe:
      httpGet:
        path: /
        port: http
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 2            
      failureThreshold: 3          
      successThreshold: 1 
    # Volumes
    volumes: []
    # Volume Mounts
    volumeMounts: []

  backend:
    # This will set the replicaset count
    replicaCount: 1
    # This sets the container image
    image:
      repository: httpd
      pullPolicy: IfNotPresent
      tag: "2.4"
    # This is to override the default name
    nameOverride: "api"
    # This is for setting Kubernetes Labels to the Pod
    podLabels: {}
    # For Setting up service
    service:
      type: ClusterIP
      port: 80
    # For setting resource requirements
    resources: {}
    # Liveness and Readness probes for the pods
    livenessProbe:
      httpGet:
        path: /
        port: http
      initialDelaySeconds: 10
      periodSeconds: 10
    readinessProbe:
      httpGet:
        path: /
        port: http
      initialDelaySeconds: 5
      periodSeconds: 5
    # Volumes
    volumes: []
    # Volume Mounts
    volumeMounts: []
```
- Deployment.yaml file
```sh
{{- range $name, $cfg := .Values.components }}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-web-app.fullname" (dict "nameOverride" $cfg.nameOverride "ctx" $) }}-{{ $name }}
  labels:
    {{- include "my-web-app.labels" (dict "name" $name "ctx" $) | nindent 4 }}
spec:
  replicas: {{ $cfg.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-web-app.selectorLabels" (dict "name" $name "ctx" $) | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-web-app.labels" (dict "name" $name "ctx" $) | nindent 8 }}
        {{- with $cfg.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      containers:
        - name: {{ $name }}
          image: "{{ $cfg.image.repository }}:{{ $cfg.image.tag | default $.Chart.AppVersion }}"
          imagePullPolicy: {{ $cfg.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ $cfg.service.port }}
              protocol: TCP
          {{- with $cfg.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with $cfg.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with $cfg.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with $cfg.volumeMounts }}
          volumeMounts:
            {{- toYaml . | nindent 12 }}
          {{- end }}
      {{- with $cfg.volumes }}
      volumes:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
```
By iterating over the keys (Frontend & backend) at values.yaml using range action, we can render multiple deployment and service yamls for different application. This reduces the number of manifest files to manage.

- Liveness and Readiness probes

The liveness probe is used to determine whether a container inside a Pod is still running in a healthy state. It runs periodically and sends requests to a configured endpoint or command inside the container. If the container fails to respond correctly (e.g., times out, returns an error, or doesn't respond), Kubernetes considers the container unhealthy. After a configurable number of consecutive failures (failure threshold), Kubernetes will automatically restart the container. This helps recover from situations like deadlocks, memory leaks, etc. Thus, liveness probes are essential for maintaining long-term application health in a production environment.

The readiness probe is used to determine whether a Pod is ready to accept traffic. When a Pod starts, Kubernetes uses the readiness probe to check if the application has fully initialized. If the readiness probe fails, the Pod is excluded from the Service's endpoints, meaning no traffic is routed to it. This prevents clients from receiving errors due to incomplete initialization of an application. Readiness probes are also important during rolling updates. Kubernetes ensures that new Pods are marked as “ready” before sending traffic to them, while old Pods continue to serve traffic until the new ones are healthy. This transition helps avoid downtime or user-facing errors during deployment.

### Part 3: Deployment and Release Management
- Dry run command
```sh
$ helm install webapp-prod ./my-web-app --dry-run --debug
```
Helm dry run commands are used to render the Kubernetes manifest files without making any actual changes to the cluster. It’s a safe way to preview what will happen before applying it for real. It also helps debug template syntax errors, incorrect values, missing variables, and other logic issues in Helm charts. It also validates the helm life cycle hooks.
- Deploying Chart
```sh
$ helm upgrade --install webapp-prod ./my-web-app -n production --create-namespace
```
- Verifying

Running helm test for checking the deployed pods
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ helm test webapp-prod -n production
NAME: webapp-prod
LAST DEPLOYED: Fri Jun 20 22:17:12 2025
NAMESPACE: production
STATUS: deployed
REVISION: 1
TEST SUITE:     api-backend-test
Last Started:   Fri Jun 20 22:19:51 2025
Last Completed: Fri Jun 20 22:19:57 2025
Phase:          Succeeded
TEST SUITE:     ui-frontend-test
Last Started:   Fri Jun 20 22:19:58 2025
Last Completed: Fri Jun 20 22:20:02 2025
Phase:          Succeeded
```
Checking deployment, pods and services using kubectl commands
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get deployments -n production
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-backend   1/1     1            1           16m
ui-frontend   1/1     1            1           16m
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get pods -n production
NAME                          READY   STATUS      RESTARTS   AGE
api-backend-8f6b758dd-k5qn8   1/1     Running     0          19m
api-backend-test-4fxpj        0/1     Completed   0          17m
ui-frontend-86d4cb99d-fd5kb   1/1     Running     0          19m
ui-frontend-test-4q7lq        0/1     Completed   0          17m
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get svc -n production
NAME                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
api-backend-service   ClusterIP   34.118.232.43    <none>        80/TCP    24m
ui-frontend-service   ClusterIP   34.118.227.101   <none>        80/TCP    24m
```
also we can check the logs for the pods using the below command
```sh
kubectl logs <pod-name> -n <namespace>
```
- Release upgrade
Update chart and app versions for the new upgrade.
```sh
$ helm upgrade --install webapp-prod ./my-web-app -n production --create-namespace --set components.frontend.image.tag="1.21.6" --set components.backend.replicaCount=3
```
check the helm status to see the update status
```sh
$ helm status webapp-prod -n production
NAME: webapp-prod
LAST DEPLOYED: Fri Jun 20 23:04:07 2025
NAMESPACE: production
STATUS: deployed
REVISION: 2
```
check the helm history for the upgrade complete with status deployed at the newest Chart and App versions
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ helm history webapp-prod -n production
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Fri Jun 20 22:17:12 2025        superseded      my-web-app-0.1.1        1.0.0           Install complete
2               Fri Jun 20 23:04:07 2025        deployed        my-web-app-0.1.2        1.1.0           Upgrade complete
```

kubectl command for upgrade verification and then verify the increased number of pods and its status as Running
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get deployment -n production
NAME          READY   UP-TO-DATE   AVAILABLE   AGE
api-backend   3/3     3            3           57m
ui-frontend   1/1     1            1           57m
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get pod -n production
NAME                           READY   STATUS      RESTARTS   AGE
api-backend-69454489bd-bhgj5   1/1     Running     0          12m
api-backend-69454489bd-c29g8   1/1     Running     0          11m
api-backend-69454489bd-svgqz   1/1     Running     0          11m
api-backend-test-n9llg         0/1     Completed   0          36m
ui-frontend-5f6d8f7f7b-ct6f6   1/1     Running     0          12m
ui-frontend-test-4st54
```
kubectl command to verify the image version update for the frontend pods
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ kubectl get pod ui-frontend-5f6d8f7f7b-ct6f6 -o jsonpath="{.spec.containers[*].image}" -n production
nginx:1.21.6
```
- Rollback the release
command to rollback the release to a previous revision
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ helm rollback webapp-prod 1 -n production
Rollback was a success! Happy Helming!
```
verifying rollback to a previous version chart and description of Rollback
```sh
skirans049@cloudshell:~ (neat-striker-462705-a9)$ helm history webapp-prod -n production
REVISION        UPDATED                         STATUS          CHART                   APP VERSION     DESCRIPTION     
1               Fri Jun 20 22:17:12 2025        superseded      my-web-app-0.1.1        1.0.0           Install complete
2               Fri Jun 20 23:04:07 2025        superseded      my-web-app-0.1.2        1.1.0           Upgrade complete
3               Fri Jun 20 23:24:07 2025        deployed        my-web-app-0.1.1        1.0.0           Rollback to 1
```
- Secrets Management
To manage Kubernetes secrets, I chose to store secret data in an encrypted file using the SOPS tool. In environments without a full-fledged CI/CD system or external secrets manager, file-based encryption with SOPS is a simple and effective way to securely manage sensitive values such as API keys or passwords.
- Installation at Gcloud shell
```sh
$ curl -LO https://github.com/getsops/sops/releases/download/v3.10.2/sops-v3.10.2.linux.amd64
$ mv sops-v3.10.2.linux.amd64 /usr/local/bin/sops
$ chmod +x /usr/local/bin/sops
$ gcloud kms keyrings create sops --location global
$ gcloud kms keys create sops-key --location global --keyring sops --purpose encryption
$ gcloud kms keys list --location global --keyring sops
```
- Create values.secrets.yaml which contains the secret API KEY
```sh
apiKey: <sceretval>
```
- Encrypt the values.secrets.yaml and name it as values.secrets.enc.yaml
```sh
$ sops encrypt --gcp-kms projects/my-project/locations/global/keyRings/sops/cryptoKeys/sops-key values.secrets.yaml > values.secrets.enc.yaml
```
remove the plain the text yaml file for security reasons. Place the encrypted file in the secret folder within the chart.

- Create the Kubernetes secret manifest backend-secret.yaml at the templates folder
```sh
cat <<EOF > templates/backend-secret.yaml
apiVersion:v1
kind: Secret
metadata:
  name: backend-api-key
type: Opaque
data:
  API_KEY: {{ .Values.apiKey | b64enc }}
EOF
```
Modify the deployment yaml and values.yaml to configure the env variable

- Install helm-secrets plugin which will sops to decrypt the encrypted files on the fly while deploying applications using helm upgrade command
```sh
$ helm plugin install https://github.com/jkroepke/helm-secrets
```
- Run helm upgrade secrets command to decrypt the values.secrets.enc.yaml file and add the API_KEY as Kubernetes secrete object which is passed as env variable to the backend application.
```sh
$ helm secrets upgrade --install webapp-prod ./my-web-app -n production --create-namespace -f ./my-web-app/secrets/values.secrets.enc.yml
```
- Verify the helm upgrade status, pod status and secret is created successfully
```sh
$ kubectl get pods -n production
$ kubectl get secrets -n production
$ kubectl describe secrets backend-api-key -n production
```
- To check the API_KEY env variable exec into the pod and run printenv
```sh
$ kubectl exec my-backend-pod-7c9fdbffb9-8xz8d -n production -- printenv
```