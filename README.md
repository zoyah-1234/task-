Step-by-Step GitLab CI/CD Pipeline Setup
1. Project Structure (Laravel Dockerized)
bash
Copy
Edit
your-laravel-project/
├── .gitlab-ci.yml
├── Dockerfile
├── docker-compose.yml (for local)
├── k8s/ or ecs/ (infra deployment manifests)
└── ...
 2. Dockerfile (Multi-stage for Laravel)
dockerfile
Copy
Edit
# Build stage
FROM composer:2 as build

WORKDIR /app
COPY . .
RUN composer install --no-dev --optimize-autoloader

# Runtime stage
FROM php:8.2-fpm-alpine

RUN docker-php-ext-install pdo pdo_mysql

WORKDIR /var/www/html
COPY --from=build /app /var/www/html

CMD ["php-fpm"]
3. GitLab CI/CD (.gitlab-ci.yml)
yaml
Copy
Edit
stages:
  - test
  - build
  - deploy

variables:
  IMAGE_NAME: registry.gitlab.com/$CI_PROJECT_PATH/app
  TAG: $CI_COMMIT_SHORT_SHA

# -------- TEST STAGE ----------
test:
  stage: test
  image: php:8.2
  script:
    - apt update && apt install -y unzip git zip
    - curl -sS https://getcomposer.org/installer | php
    - php composer.phar install
    - php artisan test

# -------- BUILD & PUSH STAGE ----------
docker-build:
  stage: build
  image: docker:20
  services:
    - docker:20-dind
  before_script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" $CI_REGISTRY
  script:
    - docker build -t $IMAGE_NAME:$TAG .
    - docker push $IMAGE_NAME:$TAG
  only:
    - main

# -------- DEPLOY STAGE (to ECS or Kubernetes) ----------
deploy:
  stage: deploy
  image: amazon/aws-cli
  script:
    - aws sts get-caller-identity
    - echo "Deploying to ECS..."
    - aws ecs update-service --cluster your-cluster-name \
        --service your-service-name \
        --force-new-deployment
  only:
    - main
4. Secrets Management (AWS SSM / Vault)
a. Using AWS SSM
Store your .env or database credentials as parameters:

bash
Copy
Edit
aws ssm put-parameter --name /laravel/DB_PASSWORD \
  --value "supersecret" --type SecureString
In your app:

php
Copy
Edit
'password' => env('DB_PASSWORD', exec('aws ssm get-parameter --name /laravel/DB_PASSWORD --with-decryption --query Parameter.Value --output text')),
b. Using Vault (Optional Advanced)
Store secrets in HashiCorp Vault, and inject them into GitLab runner via environment variables or CI/CD environment secrets.

 5. Kubernetes or ECS Deployment
a. ECS
Use aws ecs update-service as above.

b. Kubernetes
If deploying to EKS or K8s:

yaml
Copy
Edit
deploy:
  stage: deploy
  image: bitnami/kubectl
  script:
    - kubectl set image deployment/laravel-deploy app=$IMAGE_NAME:$TAG
    - kubectl rollout status deployment/laravel-deploy
