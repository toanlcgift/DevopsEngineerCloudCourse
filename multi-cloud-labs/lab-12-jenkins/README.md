# Lab 12 – CI/CD Pipeline with Jenkins

Run Jenkins locally and execute a pipeline that builds and tests a container image. For this isolated learning exercise, Jenkins uses the host Docker socket. Access to that socket is effectively root access to the WSL Docker host; never use this setup for untrusted jobs or production. Complete the [WSL and Docker setup](../README.md#start-here).

## 1. Create a Jenkins image and Compose service

```bash
mkdir -p ~/devops-labs/lab12
cd ~/devops-labs/lab12
cat > Dockerfile.jenkins <<'EOF'
FROM jenkins/jenkins:lts-jdk21
USER root
RUN apt-get update && apt-get install -y --no-install-recommends docker.io \
    && rm -rf /var/lib/apt/lists/*
USER jenkins
EOF

cat > compose.yaml <<'EOF'
services:
  jenkins:
    build:
      context: .
      dockerfile: Dockerfile.jenkins
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:50000:50000"
    group_add:
      - "${DOCKER_GID:?Set DOCKER_GID to the Docker socket group ID}"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
volumes:
  jenkins_home:
EOF
```

## 2. Add a sample application, image, and pipeline

```bash
cat > app.py <<'EOF'
print("Jenkins built and ran the lab application")
EOF

cat > Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
EOF

cat > Jenkinsfile <<'EOF'
pipeline {
  agent any
  stages {
    stage('Prepare sample') {
      steps {
        writeFile file: 'app.py', text: 'print("Jenkins built and ran the lab application")\n'
        writeFile file: 'Dockerfile', text: 'FROM python:3.12-slim\nWORKDIR /app\nCOPY app.py .\nCMD ["python", "app.py"]\n'
      }
    }
    stage('Build image') {
      steps { sh 'docker build -t lab12-app:${BUILD_NUMBER} .' }
    }
    stage('Test image') {
      steps { sh 'docker run --rm lab12-app:${BUILD_NUMBER}' }
    }
  }
  post {
    always { sh 'docker image rm lab12-app:${BUILD_NUMBER}' }
  }
}
EOF
```

## 3. Start Jenkins and create a Pipeline job

```bash
export DOCKER_GID=$(stat -c '%g' /var/run/docker.sock)
docker compose up -d --build
docker compose logs --tail 50 jenkins
docker compose exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

Open `http://127.0.0.1:8080` in a browser, unlock Jenkins with the displayed one-time password, install suggested plugins, and create a **Pipeline** job. Choose **Pipeline script** and paste the contents of `Jenkinsfile`. Save and run **Build Now**. Verify the prepare, build, and test stages succeed.

Do not expose Jenkins to a public interface or connect a production Docker host. Never store real cloud credentials in a Jenkinsfile; use a managed secret store and short-lived identity in real pipelines.

## 4. Shut down and remove Jenkins

```bash
docker compose down -v
unset DOCKER_GID
```

The `-v` option deletes Jenkins configuration, jobs, and build history.
