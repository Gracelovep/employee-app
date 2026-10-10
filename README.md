Employee Directory App
A web application to manage employees — built with Docker and deployed on AWS EKS.

Part 1 — Launch an EC2 Instance
Go to EC2 → Launch Instance

Choose:

AMI: Amazon Linux 2023
Instance type: t3.medium (2 vCPU, 4 GB RAM)
Storage: 20 GB gp3
Create or select a key pair (.pem file)

Configure Security Group — open these ports:

Port	Purpose
22	SSH
80	HTTP
5000	Backend API
8080	Frontend
Launch and SSH in:

chmod 400 your-key.pem
ssh -i your-key.pem ec2-user@<EC2_PUBLIC_IP>
Part 2 — Install Docker on the EC2 Instance
sudo dnf update -y
sudo dnf install -y docker git
sudo systemctl enable --now docker
sudo usermod -aG docker ec2-user
newgrp docker
Verify:

docker --version
Part 3 — Clone the Repo
git clone https://github.com/LandmakTechnology/employee-app.git
cd employee-app
Part 4 — Build the Docker Images
docker build -t employee-backend:v1 ./backend
docker build -t employee-frontend:v1 ./frontend
Verify:

docker images
Part 5 — Tag and Push to DockerHub
Log in to DockerHub:
docker login
# enter your DockerHub username and password/token
Tag the images (replace <your-dockerhub-username>):
docker tag employee-backend:v1 <your-dockerhub-username>/employee-backend:v1
docker tag employee-frontend:v1 <your-dockerhub-username>/employee-frontend:v1
Push:
docker push <your-dockerhub-username>/employee-backend:v1
docker push <your-dockerhub-username>/employee-frontend:v1
Part 6 — Install Terraform on Your Local Machine
macOS (Homebrew):

brew tap hashicorp/tap
brew install hashicorp/tap/terraform
Linux:

sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install terraform
Verify:

terraform -version
Part 7 — Create an IAM User and Configure AWS CLI
7.1 Create the IAM User
Go to IAM → Users → Create user
Username: terraform-user
Select Attach policies directly and attach:
AmazonEKSClusterPolicy
AmazonEKSWorkerNodePolicy
AmazonEC2FullAccess
IAMFullAccess
AmazonVPCFullAccess
AmazonEKSFullAccess (or AdministratorAccess for a lab environment)
Go to the user → Security credentials → Create access key
Choose CLI use case and save the Access Key ID and Secret Access Key
7.2 Install AWS CLI
macOS:

brew install awscli
Linux:

curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
7.3 Configure AWS CLI
aws configure --profile terraform
Enter when prompted:

AWS Access Key ID:     <your-access-key-id>
AWS Secret Access Key: <your-secret-access-key>
Default region name:   us-east-1
Default output format: json
Verify:

aws sts get-caller-identity --profile terraform
Part 8 — Create the Kubernetes Cluster on AWS with Terraform
cd employee-app/terraform
Initialize Terraform:

terraform init
Preview the infrastructure:

terraform plan -var-file=env/dev/terraform.tfvars
Apply (this creates the VPC, EKS cluster, and node group — takes ~15 min):

terraform apply -var-file=env/dev/terraform.tfvars
Type yes when prompted.

Part 9 — Authenticate kubectl to the EKS Cluster
Install kubectl if not already installed:

# macOS
brew install kubectl

# Linux
curl -LO "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/
Update your kubeconfig:

aws eks update-kubeconfig --name landmark-cluster-dev --region us-east-1 --profile terraform
Verify the connection:

kubectl get nodes
Part 10 — Deploy a Pod, Service, and Deployment Using Your DockerHub Image
10.1 Create the Namespace
kubectl create namespace employee-app
10.2 Create a Pod
kubectl run backend-pod \
  --image=<your-dockerhub-username>/employee-backend:v1 \
  --port=5000 \
  --namespace=employee-app
Check it:

kubectl get pods -n employee-app
kubectl logs backend-pod -n employee-app
10.3 Create a Deployment
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: employee-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: <your-dockerhub-username>/employee-backend:v1
          ports:
            - containerPort: 5000
Apply:

kubectl apply -f deployment.yaml
kubectl rollout status deployment/backend -n employee-app
10.4 Create a NodePort Service
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
  namespace: employee-app
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 5000
      targetPort: 5000
      nodePort: 30500
Apply:

kubectl apply -f service.yaml
kubectl get svc -n employee-app
Part 11 — Access the App via NodePort
Get the public IP of any worker node:
kubectl get nodes -o wide
# copy the EXTERNAL-IP of any node
Make sure port 30500 is open in the EC2 Security Group of the worker nodes.

Open in your browser or curl:

curl http://<NODE_EXTERNAL_IP>:30500/api/health
API Endpoints
Method	Path	Description
GET	/api/employees	List all employees
POST	/api/employees	Create employee
PUT	/api/employees/<id>	Update employee
DELETE	/api/employees/<id>	Delete employee
GET	/api/health	Health check
About
No description, website, or topics provided.
Resources
Readme
Activity
Custom properties
Stars
1 star
Watchers
0 watching
Forks
18 forks
Report repository
Releases
No releases published
Deployments
137
 (137)
development
production
Packages
No packages published
Contributors
2
 (2)
@CHAFAH
CHAFAHChafah Sani
Languages
HTML
64%
HCL
9.4%
Python
7.7%
Java
7.3%
JavaScript
6.1%
Shell
4.6%
Other
0.9%
Footer
