Create a developer server, add ssh key to the github account, clone project files, push files to the repository.

Create Access-Secret key for AWS policies Go to repository settings > secrets and variables > actions >

image

Create new ecr repository in aws : mansi-30

Create eks-server and install kubectl and eksctl

ssh-keygen
aws-configure
eksctl create cluster --name my-cluster --region ap-southeast-1 --version 1.29 --vpc-public-subnets subnet-044c44fef8814b390,subnet-065909e8f402e86a7 --node-type t2.micro --nodes-min 2 --ssh-access --ssh-public-key /root/.ssh/id_rsa.pub
mkdir manifest
cd manifest
vim deployment.yml
For deployment.yml paste:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app1
  namespace: default
  labels:
    app: my-app1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app1
  template:
    metadata:
      labels:
        app: my-app1
    spec:
      containers:
      - name: my-app1
        image: 288761750357.dkr.ecr.ap-southeast-1.amazonaws.com/mansi-30:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
vim service.yml
For service.yml paste:

apiVersion: v1
kind: Service
metadata:
  name: regapp-service1
  labels:
    app: my-app1
spec:
  selector:
    app: my-app1

  ports:
    - port: 8080
      targetPort: 8080

  type: LoadBalancer
In github repository create .github/workflows/deploy.yml and paste:
name: Deploy to ECR

on: 
  push:
    branches: [ main ]

env:
  ECR_REPOSITORY: mansi-30
  EKS_CLUSTER_NAME: my-cluster
  AWS_REGION: ap-southeast-1

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
        kubectl rollout restart deployment/my-app1
        kubectl apply -f deployment.yml
        kubectl apply -f service.yml
and run the file
