Deploying a Ruby on Rails application via Capistrano versus using DevOps tools like AWS EKS (Kubernetes) or KIND (Kubernetes IN Docker) represents a shift in architecture, deployment style, and operational responsibility.

Below is a comparison and a migration guide from Capistrano (traditional server-based) to containerized environments like EKS or KIND.

🔍 Comparison: Capistrano vs AWS EKS vs KIND

| Feature/Aspect               | **Capistrano**                      | **AWS EKS (Kubernetes)**                      | **KIND (for local K8s)**                        |
| ---------------------------- | ----------------------------------- | --------------------------------------------- | ----------------------------------------------- |
| **Deployment Style**         | SSH-based, pull from Git            | Push Docker images, deploy via manifests/Helm | Push Docker images, deploy via kubectl          |
| **Architecture**             | Single-node or manually provisioned | Clustered, auto-scaled, microservices         | Simulated cluster inside Docker for local dev   |
| **Environment**              | Bare metal / VM / VPS               | Fully managed cloud-native infrastructure     | Local environment for testing Kubernetes setups |
| **Scaling**                  | Manual (vertical or horizontal)     | Automatic (HPA, load balancer)                | Not scalable (test only)                        |
| **Zero Downtime Deployment** | Not default, requires tuning        | Native with rolling updates                   | Manual but testable                             |
| **CI/CD Integration**        | Limited, manual scripts or triggers | Easily integrates with GitHub Actions, ArgoCD | Supports same tools as EKS (but for dev only)   |
| **Rollback**                 | Easy with `cap rollback`            | Handled via `kubectl rollout undo` or Helm    | Same as EKS (manual or scripted)                |
| **Secrets Management**       | Shared files (e.g., `master.key`)   | Native (AWS Secrets Manager, K8s Secrets)     | Same as Kubernetes                              |
| **Monitoring/Logs**          | Manual (e.g., tail logs on server)  | Integrated with CloudWatch, Prometheus, etc.  | Use `kubectl logs` or local logging tools       |
| **Usage**                    | Simple, great for monoliths         | Best for microservices, scalable platforms    | Local-only, fast iteration cycles               |


🧭 Migrating from Capistrano to Kubernetes (EKS or KIND)

# 1. Containerize Your Rails App

You need to run your Rails app as a Docker container.

Dockerfile:

```bash
FROM ruby:3.2
ENV RAILS_ENV=production # Set environment
RUN apt-get update -qq && apt-get install -y nodejs postgresql-client yarn # Install dependencies
WORKDIR /app
COPY . /app
RUN bundle install --without development test
RUN yarn install
RUN bundle exec rake assets:precompile
CMD ["bundle", "exec", "puma", "-C", "config/puma.rb"]
```

# 2. Build & Push Docker Image

docker build -t myapp .
docker tag myapp <your-dockerhub-or-ecr-repo>/myapp:latest
docker push <your-dockerhub-or-ecr-repo>/myapp:latest

# 3. Set Up Kubernetes Manifests

Deployment (deployment.yaml)

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: rails
  template:
    metadata:
      labels:
        app: rails
    spec:
      containers:
      - name: rails
        image: <your-docker-repo>/myapp:latest
        ports:
        - containerPort: 3000
        env:
        - name: RAILS_ENV
          value: production
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: rails-secrets
              key: database-url
```

Service (service.yaml)

```bash
apiVersion: v1
kind: Service
metadata:
  name: rails-service
spec:
  type: LoadBalancer
  selector:
    app: rails
  ports:
  - port: 80
    targetPort: 3000
```

# 4. Use KIND for Local Testing (Optional)

```bash
kind create cluster --name rails-cluster
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl port-forward svc/rails-service 8080:80
```

Now visit http://localhost:8080 to test.

# 5. Deploy to AWS EKS
	1.	Create EKS Cluster via AWS Console or Terraform.
	2.	Configure kubectl for EKS:
```bash
aws eks update-kubeconfig --region <region> --name <cluster-name>
```

	3.	Apply Manifests:
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

	4.	Use AWS Secrets Manager or Kubernetes Secrets for your secrets.


# 6. CI/CD Setup

Use GitHub Actions or GitLab CI to:
	•	Build Docker image
	•	Push to DockerHub or ECR
	•	Deploy to EKS using kubectl or Helm

Example GitHub Actions snippet:

```bash
name: CI/CD: Deploy Rails App to AWS EKS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  EKS_CLUSTER_NAME: ${{ secrets.EKS_CLUSTER_NAME }}
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: 📦 Checkout Code
      uses: actions/checkout@v4

    - name: 🔐 Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: 🔑 Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v2

    - name: 🐳 Build Docker Image
      run: |
        docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .

    - name: 🚀 Push Docker Image to ECR
      run: |
        docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG

    - name: 📡 Update Kubeconfig
      run: |
        aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME

    - name: 🔧 Apply K8s Manifests (Deployment, Service, Secrets)
      run: |
        kubectl apply -f k8s/deployment.yaml
        kubectl apply -f k8s/service.yaml
        kubectl apply -f k8s/secrets.yaml

    - name: 🔄 Run DB Migrations via Kubernetes Job
      run: |
        # Define a unique job name to avoid conflict
        MIGRATION_JOB_NAME=rails-db-migrate-${{ github.run_id }}

        # Create the Job dynamically from the deployment template
        kubectl create job $MIGRATION_JOB_NAME --from=deployment/rails-app

        # Wait for it to complete (max 180s)
        kubectl wait --for=condition=complete --timeout=180s job/$MIGRATION_JOB_NAME

        # Delete the Job after completion
        kubectl delete job $MIGRATION_JOB_NAME

    - name: 🚢 Deploy Latest Image to EKS
      run: |
        kubectl set image deployment/rails-app rails=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
        kubectl rollout status deployment/rails-app

```

🏁 Final Thoughts

Migration Summary
Capistrano is fast and great for legacy Rails apps and monoliths.
Kubernetes (EKS) is ideal for scalable, containerized Rails apps in microservice or cloud-native setups.
KIND is for local testing only — don’t use it in production.
Moving to EKS requires Dockerizing your app and replacing Capistrano scripts with K8s YAML or helm charts
