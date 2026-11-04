# Git Collaboration and Jenkins CI

## Reviewable Git Collaboration

```bash
echo "Step 1/3: Create a feature branch and add a health file"
cd ~/devops-labs/hello-devops
git switch -c feature/add-health-file
printf 'ok\n' > app/health.txt
git add app/health.txt
git commit -m "Add health marker"
git show --stat --oneline HEAD
echo "Step 2/3: Merge the feature branch"
git switch main
git merge --no-ff feature/add-health-file
echo "Step 3/3: Remove the merged branch"
git branch -d feature/add-health-file
```

**Discussion:** Compare this Git Flow-style feature branch with trunk-based development. Which automated checks should block a merge? In a shared team repo, use a remote pull/merge request and review instead of merging directly.

**Debug:** If a branch or merge is not as expected, use `git status`, `git branch --verbose`, and `git log --oneline --graph --decorate --all`. Resolve any conflict markers before committing.

## Jenkins Pipeline: Test the Application

Add a simple test script and `Jenkinsfile` to the repository:

```bash
echo "Step 1/3: Create the test script"
cd ~/devops-labs/hello-devops
cat > test.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
test -s app/index.html
grep -q '<h1>' app/index.html
echo "Tests passed"
EOF
chmod +x test.sh
echo "Step 2/3: Create the Jenkins pipeline"
cat > Jenkinsfile <<'EOF'
pipeline {
  agent any
  stages {
    stage('Test') {
      steps { sh 'cd /workspace && ./test.sh' }
    }
  }
}
EOF
echo "Step 3/3: Commit the pipeline and run the test locally"
git add test.sh Jenkinsfile
git commit -m "Add Jenkins test pipeline"
./test.sh
```

Start a local Jenkins controller for the lab:

```bash
echo "Step 1/2: Create persistent Jenkins storage and start Jenkins"
docker volume create jenkins_home
docker run -d --name jenkins --restart unless-stopped \
  -p 127.0.0.1:8080:8080 -p 127.0.0.1:50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v "$HOME/devops-labs/hello-devops:/workspace:ro" \
  jenkins/jenkins:lts-jdk17
echo "Step 2/2: Show recent Jenkins startup logs"
docker logs --tail 50 jenkins
```

The container keeps running. Read the initial admin password and open `http://127.0.0.1:8080` through an SSH tunnel if your browser is on another machine:

```bash
echo "Jenkins initial admin password:"
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
# On your workstation, in a separate terminal:
ssh -L 8080:127.0.0.1:8080 <ubuntu-user>@<ubuntu-host>
```

Complete Jenkins' one-time setup, install suggested plugins (including Pipeline), and create a Pipeline job. Use **Pipeline script** and paste the contents of `Jenkinsfile`. The read-only bind mount makes the lab repository available to the job at `/workspace`. In a team setup, configure **Pipeline script from SCM** and use a remote Git repository instead.

The Jenkins UI setup is deliberately a one-time bootstrap; the pipeline definition and application checks stay in files and can be version controlled. Do not expose this unauthenticated lab controller publicly. Do not mount the host Docker socket into Jenkins on a shared or production host: it grants broad control of the host.

**Verify:** Jenkins reports the Test stage as successful. Change the test to fail temporarily, run it again, and observe that the pipeline fails; restore the passing test afterwards.

**Debug:** Inspect controller startup and job output with `docker logs --tail 100 jenkins` and the Jenkins job's **Console Output**. Check container health/state with `docker ps -a --filter name=jenkins` and verify the mounted files with `docker exec jenkins ls -la /workspace`. The job console identifies the failing stage and command.

---
