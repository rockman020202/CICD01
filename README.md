# 1. Create ECR

## 1.1 Command

```bash
aws ecr create-repository --region us-east-1 --repository-name <APPNAME> --image-scanning-configuration scanOnPush=true
```

## 1.2 Sample

```bash
aws ecr create-repository --region us-east-1 --repository-name fbvn-hunght-docker --image-scanning-configuration scanOnPush=true
```

```json
{
  "repository": {
    "repositoryArn": "arn:aws:ecr:us-east-1:624564778233:repository/fbvn-hunght-docker",
    "registryId": "624564778233",
    "repositoryName": "fbvn-hunght-docker",
    "repositoryUri": "624564778233.dkr.ecr.us-east-1.amazonaws.com/fbvn-hunght-docker",
    "createdAt": "2025-07-22T21:30:08.491000+07:00",
    "imageTagMutability": "MUTABLE",
    "imageScanningConfiguration": { "scanOnPush": true },
    "encryptionConfiguration": { "encryptionType": "AES256" }
  }
}
```

# 2. Build Docker & Login to ECR

## 2.1 Command

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com
```

## 2.2 Sample

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 624564778233.dkr.ecr.us-east-1.amazonaws.com
```

# 3. Build, Tag & Push Image

## 3.1 Commands

```bash
docker build -t <my-app>:latest .
docker tag <my-app>:latest <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/<my-app>:latest
docker push <AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/<my-app>:latest
```

## 3.2 Sample

```bash
docker build -t fbvn-hunght-docker:latest .
docker tag fbvn-hunght-docker:latest 624564778233.dkr.ecr.us-east-1.amazonaws.com/fbvn-hunght-docker:latest
docker push 624564778233.dkr.ecr.us-east-1.amazonaws.com/fbvn-hunght-docker:latest
```

# 4. Create ECS Cluster

## 4.1 Command

```bash
aws ecs create-cluster --cluster-name <my-app-cluster>
```

## 4.2 Sample

```bash
aws ecs create-cluster --cluster-name fbvn-hunght-docker-cluster
```

# 5. Task Definition

## 5.1 Create `taskdef.json`

### 5.1.1 Template

```json
{
  "family": "my-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::<AWS_ACCOUNT_ID>:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "<my-app>",
      "image": "<AWS_ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com/<my-app>:latest",
      "portMappings": [
        { "containerPort": 80, "protocol": "tcp" }
      ],
      "essential": true
    }
  ]
}
```

### 5.1.2 Sample

```json
{
  "family": "fbvn-hunght-docker-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::624564778233:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "fbvn-hunght-docker",
      "image": "624564778233.dkr.ecr.us-east-1.amazonaws.com/fbvn-hunght-docker:latest",
      "portMappings": [
        { "containerPort": 80, "protocol": "tcp" }
      ],
      "essential": true
    }
  ]
}
```

## 5.2 Register Task Definition

```bash
aws ecs register-task-definition --cli-input-json file://taskdef.json
```

# 6. Create Service

## 6.1 Command

```bash
aws ecs create-service --cluster <my-app-cluster> --service-name <my-app>-service --task-definition <my-app>-task --desired-count 2 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-XXX],securityGroups=[sg-YYY],assignPublicIp=ENABLED}"
```

## 6.2 Sample

```bash
aws ecs create-service --cluster fbvn-hunght-docker-cluster --service-name fbvn-hunght-docker-service --task-definition fbvn-hunght-docker-task --desired-count 2 --launch-type FARGATE --network-configuration "awsvpcConfiguration={subnets=[subnet-0d1a9bcff04414937],securityGroups=[sg-050647e358328a7fe],assignPublicIp=ENABLED}"
```

> Thay `subnet-XXX` và `sg-YYY` bằng giá trị thực.

# 7. Edit `buildspec.yml`

* Cập nhật các biến môi trường (`AWS_DEFAULT_REGION`, `REPO_URI`).
* Thêm các lệnh login, build, push và xuất `imagedefinitions.json`.

# 8. Push Source lên Git

```bash
git add . && git commit -m "Add buildspec.yml and ECS configs" && git push origin main
```

# 9. Create CodePipeline

* **Source Stage**: Chọn CodeCommit/GitHub.
* **Build Stage**:

  * Action type: AWS CodeBuild
  * Project: sử dụng `buildspec.yml`.
* **Deploy Stage**:

  * Action type: Amazon ECS
  * Cluster: `my-app-cluster`
  * Service: `my-app-service`
  * Image definitions file: `imagedefinitions.json`
<img width="1381" height="621" alt="image" src="https://github.com/user-attachments/assets/d9fa9722-8d48-478c-b626-ac746aa374a8" />

# *** Force Deploy

```bash
aws ecs update-service --cluster fbvn-hunght-docker-cluster --service-name fbvn-hunght-docker-service --force-new-deployment
```
# *** Deploy with AWS App Runner

1. In the AWS Console, navigate to **App Runner** and click **Create service**.
2. Under **Source**, select **Container registry** → **Amazon ECR**, then choose your repository and image tag (e.g., `latest`).
3. In **Deployment settings**, enable **Auto-deploy** on image update.
4. Under **Service settings**:

   * **Port**: `80` (or your application port)
   * **Instance size**: choose the desired CPU/memory configuration
   * **Health check path**: `/`
   * **Networking**: set to **Public**
   * **Environment variables**: add any required variables
   * **Tags**: optional
5. Click **Create & deploy**.

Once the service is active, App Runner will provide a default endpoint, for example:

```text
https://abcdefg123.us-east-1.awsapprunner.com
```

Paste this URL into your browser to view your live application.

**Auto-deploy:** Whenever your CI/CD pipeline pushes a new image to ECR, App Runner will automatically pull and deploy the updated image.

## Custom domain (optional)

* In the App Runner console, open your service and go to **Custom domains**.
* Click **Add domain**, select your Route 53 hosted zone, and follow the wizard to configure your custom domain.
  <img width="1568" height="244" alt="image" src="https://github.com/user-attachments/assets/e589302e-38fe-48fd-a042-6793c296868e" />

# Check result
https://gamqwzptme.us-east-1.awsapprunner.com/WeatherForecast
<img width="2374" height="189" alt="image" src="https://github.com/user-attachments/assets/6b7335c2-f419-4e46-9a85-d952f08291d2" />


