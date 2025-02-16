# RBAC_Assignment

## Assignment 1 (With Minikube)

### Step 1: Start Minikube
```
minikube start
kubectl cluster-info #check status
```

### Step 2: Create Admin and Read-Only Roles
Kubernetes already has a built-in ClusterRole called cluster-admin. We’ll use that for the admin user.
Save the following as read-only-role.yaml
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
```

### Step 3: Create Admin and Baby Users
Kubernetes doesn’t manage normal users directly — but you can simulate users with certificates.

#### 3.1 Generate Certificates
Create a private key for each user:
```
openssl genrsa -out admin.key 2048
openssl genrsa -out baby.key 2048

Create certificate signing requests (CSRs):
openssl req -new -key admin.key -out admin.csr -subj "/CN=admin"
openssl req -new -key baby.key -out baby.csr -subj "/CN=baby"

Sign the CSRs with Minikube’s CA:
minikube ssh "sudo cat /var/lib/minikube/certs/ca.crt" > ca.crt
minikube ssh "sudo cat /var/lib/minikube/certs/ca.key" > ca.key
ls -l ca.crt ca.key
openssl x509 -req -in admin.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out admin.crt -days 365
openssl x509 -req -in baby.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out baby.crt -days 365
```
### Step 4: Configure Kubernetes Users
#### 4.1 Add Users to Kubeconfig
```
kubectl config set-credentials admin \
  --client-certificate=admin.crt \
  --client-key=admin.key

kubectl config set-credentials baby \
  --client-certificate=baby.crt \
  --client-key=baby.key

kubectl config set-context admin-context --cluster=minikube --user=admin
kubectl config set-context baby-context --cluster=minikube --user=baby
```
### Step 5: Bind Roles to Users
Create a ClusterRoleBinding for admin:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
subjects:
- kind: User
  name: admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
Create a ClusterRoleBinding for baby:
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: baby-binding
subjects:
- kind: User
  name: baby
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: read-only
  apiGroup: rbac.authorization.k8s.io
```
### Step 6: Test the Access
#### 6.1 Test as Admin
```
kubectl config use-context admin-context
kubectl get pods --all-namespaces
kubectl create namespace test-ns
kubectl delete namespace test-ns
```
#### 6.2 Test as Baby
```
kubectl config use-context baby-context
kubectl get pods -A # ya,u can
kubectl create namespace test-ns #Forbidden!
```