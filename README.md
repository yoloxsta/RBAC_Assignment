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
## Assignment-2
Steps for Setting Up User Access for baby on Another Machine
### Step 1: Generate User Certificates for baby

1. On your admin machine, create a private key and CSR (Certificate Signing Request) for baby:
```
openssl genrsa -out baby.key 2048
openssl req -new -key baby.key -out baby.csr -subj "/CN=baby"
openssl x509 -req -in baby.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out baby.crt -days 365

## Now you have the following files:
1.baby.key (private key)
2.baby.crt (signed certificate)
3.ca.crt (CA certificate)
```
### Step 2: Transfer Certificates to baby Machine
```
scp baby.crt baby.key ca.crt user@baby-machine:/path/to/certs/
```
### Step 3: Configure baby's Kubeconfig on Their Machine
```

kubectl config set-credentials baby --client-certificate=/path/to/certs/baby.crt --client-key=/path/to/certs/baby.key
kubectl config set-context baby-context --cluster=minikube --user=baby
kubectl config use-context baby-context

```
### Step 4: Optional - Enable Remote Access
If your Minikube cluster is running locally and you want baby to access it from a different machine, you need to expose the Minikube API externally.

Set up the Minikube API to be externally accessible by enabling port forwarding or exposing the Kubernetes API server through an external IP. One option is to use minikube tunnel to make Minikube’s Kubernetes cluster accessible.

Run the following on the admin machine:
```
minikube tunnel #This will expose the Minikube API on an external IP (e.g., localhost:8443 or another accessible address).
```
On baby’s machine, update the kubeconfig to point to the external Minikube IP address (e.g., localhost:8443).

```
kubectl config set-cluster minikube --server=https://<minikube-ip>:8443 --certificate-authority=/path/to/certs/ca.crt
```
#### service account (medium)
https://medium.com/@subhampradhan966/implementing-and-verifying-kubernetes-service-accounts-a-step-by-step-guide-c43b727260b2

https://medium.com/@HoussemDellai/rbac-with-kubernetes-in-minikube-4deed658ea7b 

## Assignment-3 (give EC2 access for other-user)
```
 sudo adduser sta --disabled-password
 sudo su - sta
 mkdir ~/.ssh
 chmod 700 ~/.ssh
 ssh-keygen -t rsa -b 2048 -f ~/.ssh/sta_key #it Generate privatekey and publickey
 ls ~/.ssh
 chmod 700 ~/.ssh
 chmod 600 ~/.ssh/sta_key
 chmod 600 ~/.ssh/sta_key.pub
 cat ~/.ssh/sta_key.pub >> ~/.ssh/authorized_keys
 chmod 600 ~/.ssh/authorized_keys
 ls -l ~/.ssh/sta_key
 exit
 sudo cp /home/sta/.ssh/sta_key /home/ubuntu/ #from ubuntu
 sudo chmod 644 /home/ubuntu/sta_key  # Make it readable for download
 sudo chown ubuntu:ubuntu /home/ubuntu/sta_key  # Ensure the file is owned by ubuntu
 scp -i .\original.pem ubuntu@ip:/home/ubuntu/sta_key C:\download\ #from exernal terminal
 ssh -i C:\Users\User\Downloads\sta_key sta@ip

```
## Assignment-4 (SFTP)
```
 sudo apt-get update
 sudo apt-get install openssh-server
 sudo systemctl enable ssh
 sudo systemctl start ssh
 sudo systemctl status ssh
 sudo adduser sftpuser
 sudo mkdir -p /home/sftpuser/uploads
 sudo chown root:root /home/sftpuser
 sudo chmod 755 /home/sftpuser
 sudo chown sftpuser:sftpuser /home/sftpuser/uploads
 sudo nano /etc/ssh/sshd_config

Match User sftpuser
    ForceCommand internal-sftp
    PasswordAuthentication yes
    ChrootDirectory /home/sftpuser
    PermitTunnel no
    AllowAgentForwarding no
    AllowTcpForwarding no
    X11Forwarding no

```