# Deploying a Chat App on EC2 with CI/CD using GitHub Actions, Docker Compose, and Nginx

## Prerequisites

- An AWS account
- A registered domain (optional, but required for Nginx setup)
- A GitHub repository for the Chat App
- Basic knowledge of Linux and Docker

## Step 1: Launch an Ubuntu EC2 Instance

1. Go to AWS EC2 Console.
2. Click **Launch Instance**.
3. Select **Ubuntu 24.04 LTS** as the AMI.
4. Choose an instance type (e.g., `t2.small` or `t2.medium`.
5. Configure security group:
   - Allow SSH (Port 22)
   - Allow HTTP (Port 80)
   - Allow HTTPS (Port 443)
   - Allow application port (e.g., `3000` for WebSocket-based chat app)
   - Allow MongoDB port (`27017`)
6. Add a key pair and download it (`.pem` file).
7. Launch the instance.

## Step 2: Connect to Your EC2 Instance

```bash
ssh -i your-key.pem ubuntu@your-ec2-ip
```
## Step 3: Install Required Packages

```sh
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
```sh
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Add permission for current user
sudo usermod -aG docker $USER
# Apply permission for docker.sock to run docker without sudo
sudo chmod 666 /var/run/docker.sock 

# Install nginx and git 
sudo apt install -y git nginx
```
## Step 4: Clone the Chat App Repository
```sh
git clone https://{{ Token }}github.com/your-repo/chat-app.git
cd chat-app
```

## Step 5: Create a Docker Compose File (docker-compose.yml)

```sh

version: '3.8'

services:
  backend:
    build: .
    container_name: chat-backend
    restart: always
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - MONGO_URI=mongodb://mongo:27017/chatdb
    depends_on:
      - mongo

  frontend:
    build: .
    container_name: chat-frontend
    restart: always
    ports:
      - "3000:3000"
    depends_on:
      - backend

  mongo:
    image: mongo:latest
    container_name: mongo
    restart: always
    ports:
      - "27017:27017"
    volumes:
      - mongo-data:/data/db

  nginx:
    image: nginx:latest
    container_name: nginx
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend

volumes:
  mongo-data:
```

## Step 6: Configure Nginx (nginx.conf)

```sh
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://frontend:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

## Step 7: Setup CI/CD with GitHub Actions (.github/workflows/deploy.yml)


```sh
name: Deploy Chat App to EC2

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3
    
    - name: SSH into EC2 and deploy
      uses: appleboy/ssh-action@v0.1.4
      with:
        host: ${{ secrets.EC2_HOST }}
        username: ubuntu
        key: ${{ secrets.EC2_SSH_KEY }}
        script: |
          cd ~/chat-app
          git pull origin main
          docker compose down
          docker compose up -d --build
```

## Step 8: Set Up GitHub Secrets

Go to your repository Settings → Secrets and add:

- EC2_HOST: Your EC2 public IP

- EC2_SSH_KEY: Your EC2 private key

## Step 9: Start the Application
```sh
docker-compose up -d
docker-compose up -d --build # If you wanna build again 
```

## Step 10: Test Your Application

- Visit http://yourdomain.com or http://your-ec2-ip:80 in a browser

##  🎉 Congratulations! You have successfully deployed your chat app on EC2 with CI/CD.