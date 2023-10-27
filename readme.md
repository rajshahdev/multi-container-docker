# Multi-Container Docker Deployment to AWS Beanstalk

This repository provides a comprehensive setup for deploying a multi-container Docker application integrating `nginx`, a frontend React app, a backend service, and a worker. The setup is prepared for continuous integration and deployment using Travis CI to AWS Elastic Beanstalk.

## Table of Contents

- [Overview](#overview)
- [Application Services](#application-services)
- [Local Development](#local-development)
- [Continuous Integration and Deployment](#continuous-integration-and-deployment)
- [AWS Deployment Steps](#aws-deployment-steps)
- [Nginx Explanation](#nginx-explanation)
## Overview

This project is a culmination of various technologies and practices designed to provide a seamless CI/CD pipeline for a multi-container Docker application.

## Application Services

1. **Nginx:** Web server responsible for routing traffic.
2. **Frontend:** React-based front-end application.
3. **Backend:** Server handling business logic and interfacing with data stores.
4. **Worker:** A background worker service.

## Local Development

For local development and testing, execute the following command:

```bash
docker-compose up
```

## Continuous Integration and Deployment

The CI/CD process is handled by Travis CI. On pushing changes to the `main` branch:

- The React frontend app is built and tested.
- If tests pass, Docker images are built for all services.
- These images are then pushed to Docker Hub:
  - `rajshahdev/multi-client`
  - `rajshahdev/multi-nginx`
  - `rajshahdev/multi-server`
  - `rajshahdev/multi-worker`
- AWS Elastic Beanstalk then uses these images for deployment as specified in the `dockerrun.aws.json`.

## AWS Deployment Steps

### Elastic Beanstalk Application Creation

- Navigate to the AWS Management Console.
- Use `Find Services` to search for `Elastic Beanstalk`.
- Click on “Create Application”.
- Set Application Name to 'multi-docker'.
- Scroll down to Platform and select Docker.
  - Ensure "Single Instance (free tier eligible)" is selected.
  - Click the "Next" button.
  - In "Service Role", select "Use an Existing service role".
  - Verify `aws-elasticbeanstalk-service-role` has been selected for the service role (Please visit `dockerized-react-aws-beanstalk` git repo if you know how to create role).
  - Confirm `aws-elasticbeanstalk-ec2-role` is chosen for the instance profile.
  - Click "Skip to review" button and then "Submit".
  - Wait for a green checkmark under Health, indicating successful creation.

### RDS Database Creation

- Access the AWS Management Console.
- Search for `RDS` using `Find Services`.
- Click the `Create database` button and select PostgreSQL.
- Under Templates, select the Free tier option.
- Set DB Instance identifier as `multi-docker-postgres`, Master Username to `postgres`, and both Master Password and confirmation to `postgrespassword`.
- Ensure VPC is set to the default VPC.
- Under Additional Configuration, provide the Initial database name as `fibvalues`.
- Finalize by clicking `Create Database`.

### ElastiCache Redis Creation

- Go to the AWS Management Console.
- Use `Find Services` to locate `ElastiCache`.
- Under Resources, select `Redis clusters` from the Sidebar.
- Click the `Create Redis cluster` button.
- Ensure Cluster Mode is DISABLED.
- Name the cluster `multi-docker-redis` and adjust Node type to `cache.t2.micro`.
- Change the Number of Replicas to 0 (despite the warning about Multi-AZ).
- Create a new Subnet Group named `redis`.
- Progress through the next steps and finally click `Create`.

### Creating a Custom Security Group

- Navigate to the AWS Management Console.
- Search for `VPC` using `Find Services`.
- Under Security in the left sidebar, choose `Security Groups`.
- Click `Create Security Group`.
- Name it `multi-docker` with the same description.
- Ensure VPC is set to the default VPC.
- Add an inbound rule with Port Range `5432-6379` and Source as the newly created security group.
- Save these rules.

### Applying Security Groups

#### To ElastiCache

- Go to the AWS Management Console.
- Search for `ElastiCache` using `Find Services`.
- Under Resources in the Sidebar, select `Redis clusters`.
- Modify your Redis cluster and manage its security groups.
- Select the `multi-docker` group, review changes, and modify.

#### To RDS

- Access AWS Management Console.
- Locate `RDS` using `Find Services`.
- Under Databases in the Sidebar, modify your instance.
- In Connectivity, select the new `multi-docker` security group.
- Continue and confirm these modifications.

#### To Elastic Beanstalk

- Go to the AWS Management Console.
- Search for `Elastic Beanstalk` using `Find Services`.
- Choose `Environments` from the left sidebar.
- Click on `MultiDocker-env`, then `Configuration`.
- In the Instances row, click `Edit`.
- Under EC2 Security Groups, select `multi-docker`.
- Apply and confirm changes.

### Setting Environment Variables

- Navigate to the AWS Management Console.
- Search for `Elastic Beanstalk` using `Find Services`.
- Select `Environments` from the left sidebar.
- Click `MultiDocker-env` and then `Configuration`.
- Scroll to `Updates, monitoring, and logging` and edit.
- Add the necessary environment properties, like `REDIS_HOST`, `REDIS_PORT`, and the PostgreSQL details.
- Apply these changes and ensure Health displays a green checkmark.

### IAM Keys for Deployment

- Search for the "IAM Security, Identity & Compliance Service" in AWS.
- Create a new IAM user or use an existing one.
- Ensure the user has "AdministratorAccess-AWSElasticBeanstalk" permissions.
- Create and store the Access Key ID and Secret Access Key for use in Travis CI.

### Deploying the Application

- Make a small change to your `src/App.js` file.
- Commit and push these changes.
- Monitor the build status in Travis CI.
- Upon success, AWS Elastic Beanstalk will update the environment.
- Once completed, access the application via the provided external URL.

# Nginx Explanation

In our multi-container setup, we utilize Nginx in two distinct roles:

1. **Primary Nginx Reverse Proxy**:
    - **Role**: This handles the initial incoming requests and directs them to the appropriate container based on the request path.

2. **Frontend Nginx Server**:
    - **Role**: This serves the static files of our React frontend application.

## Flow of a Request to the Frontend

### 1. Initial Request

When a user sends a request to the root `/` of the application, it is initially intercepted by our primary Nginx reverse proxy, defined in `nginx/default.conf`.

```nginx
location / {
    proxy_pass http://frontend;
}
```
The above configuration directs requests to / to an internal address: `http://frontend`.

### 2. Internal Docker Routing

The address `http://frontend` is mapped to the Nginx server inside the frontend container. This mapping is established in the upstream directive:

```nginx
upstream frontend {
    server frontend:3000;
}
```
### 3. Frontend Container's Nginx

When the request arrives at the Nginx server inside the frontend container (configured in client/nginx/default.conf), it then serves the React application's static files.

```nginx
location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html;
}
```

This configuration ensures that the static files of the React app, housed in `/usr/share/nginx/html`, are served in response to the request.