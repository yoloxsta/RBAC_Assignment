## RBAC_Assignment

### Assignment 1 (With Minikube)

```
Step 1: Start Minikube

minikube start
kubectl cluster-info #check status

Step 2: Create Admin and Read-Only Roles
Kubernetes already has a built-in ClusterRole called cluster-admin. We’ll use that for the admin user.
Save the following as read-only-role.yaml

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: read-only
rules:
- apiGroups: [""]
  resources: ["*"]
  verbs: ["get", "list", "watch"]

Step 3: Create Admin and Baby Users
Kubernetes doesn’t manage normal users directly — but you can simulate users with certificates.

3.1 Generate Certificates
Create a private key for each user:

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