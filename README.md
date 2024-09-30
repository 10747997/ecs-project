1. Create a developer server, add ssh key to the github account, clone project files, push files to the repository.
2. Create Access-Secret key for AWS policies
   Go to repository settings > secrets and variables > actions >

   ![image](https://github.com/user-attachments/assets/366c57e1-af4d-4b9c-837c-ca41c00b71c7)
   
3. Create new ecr repository in aws : reddy

4. Create eks-server and install kubectl and eksctl

```yml
ssh-keygen
aws-configure
eksctl create cluster --name my-cluster1 --region ap-south-1 --version 1.29 --vpc-public-subnets subnet-0581429335818ce21,subnet-03eb869d8e7779294 --node-type t2.micro --nodes-min 2 --ssh-access --ssh-public-key /root/.ssh/id_rsa.pub
mkdir manifest
cd manifest
vim regapp-deploy.yml
```
For regapp-deploy.yml paste:
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: regapp-deployment
  namespace: default
  labels:
    app: regapp-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: regapp-deployment
  template:
    metadata:
      labels:
        app: regapp-deployment
    spec:
      containers:
      - name: regapp-deployment
        image: 147997135022.dkr.ecr.ap-south-1.amazonaws.com/reddy:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
```
```yml
vim regapp-service.yml
```
For regapp-service.yml paste:

```yml
apiVersion: v1
kind: Service
metadata:
  name: regapp-service1
  labels:
    app: regapp-deployment
spec:
  selector:
    app:regapp-deployment

  ports:
    - port: 8080
      targetPort: 8080

  type: LoadBalancer
```
5. In github repository create .github/workflows/deploy.yml and paste:

```yml
name: Deploy to ECR

on: 
  push:
    branches: [ main ]

env:
  ECR_REPOSITORY: reddy
  EKS_CLUSTER_NAME: my-cluster1
  AWS_REGION: ap-south-1

jobs:
  build:
    name: Deployment
    runs-on: ubuntu-latest

    steps:

    - name: Set short commit SHA
      id: commit
      uses: prompt/actions-commit-hash@v2

    - name: Check out code
      uses: actions/checkout@v2

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with: 
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Set up JDK 14
      uses: actions/setup-java@v1
      with:
        java-version: 14

    - name: Build project with Maven
      run: mvn -B package --file pom.xml

    - name: Login to amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1

    - name: Build, tag and Push
      id: build-image
      env:
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        ECR_REPOSITORY: ${{ env.ECR_REPOSITORY }}
        IMAGE_TAG: "latest"
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
        echo "Pushing image to ECR..."
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        echo "::set-output name=image::$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG"

    - name: Update kube config
      run: aws eks update-kubeconfig --name $EKS_CLUSTER_NAME --region $AWS_REGION

    - name: Deploy to EKS
      env: 
        ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        IMAGE_TAG: ${{ steps.commit.outputs.short }}
      run: |
        kubectl rollout restart deployment/regapp-deployment
        kubectl apply -f regapp-deploy.yml
        kubectl apply -f regapp-service.yml
```
and run the file




