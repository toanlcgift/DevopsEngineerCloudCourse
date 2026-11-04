# DevOps Workflow, Git, and Cloud Concepts

## Git Repository Workflow: Create and Merge a Feature

```bash
echo "Step 1/2: Create the repository and application files"
cd ~/devops-labs
mkdir hello-devops && cd hello-devops
git init -b main
mkdir app
cat > app/index.html <<'EOF'
<!doctype html>
<html><body><h1>Hello from the DevOps lab</h1></body></html>
EOF
cat > README.md <<'EOF'
# Hello DevOps

A tiny static site used for the five-day DevOps lab.
EOF
echo "Step 2/2: Commit the starter application"
git add README.md app/index.html
git commit -m "Create starter site"
git status --short --branch
```

**Try it:** Make a feature branch, change the page heading, commit the change, then merge it back to `main`.

```bash
echo "Step 1/4: Create a feature branch and edit the page"
git switch -c feature/update-heading
sed -i 's/Hello from the DevOps lab/Hello from my lab/' app/index.html
git diff
echo "Step 2/4: Commit the change"
git add app/index.html
git commit -m "Update page heading"
echo "Step 3/4: Merge the feature branch"
git switch main
git merge --no-ff feature/update-heading
echo "Step 4/4: Inspect commit history"
git log --oneline --graph --decorate --all
```

**Verify:** `git status` is clean and `git log` shows both the feature and merge commits. For a trunk-based variation, make short-lived branches and merge small changes frequently; avoid keeping long-running branches.

**Debug:** If a commit or merge fails, inspect `git status`, `git diff`, and `git log --oneline --graph --decorate --all`. If Git reports an unresolved conflict, open the listed files, resolve conflict markers, then `git add <file>` and complete the merge with `git commit`.

## DevOps Delivery Workflow

Write a short delivery path for this site and identify where each activity belongs:

1. Change and review source in Git.
2. Run checks and build an immutable container image.
3. Store the image in a registry.
4. Deploy to a test environment, verify health, and promote the same image.
5. Monitor service health and roll back if the release fails.

For each step, name the responsible role, an automated check, and one failure signal. Discuss how Agile feedback loops and shared Dev/Ops ownership affect the workflow.

## Cloud Service Models

Classify these example responsibilities as primarily IaaS, PaaS, or SaaS: managing a VM's OS, deploying code to a managed application runtime, and using a hosted email service. For each, identify what the customer manages and what the provider manages.

Cloud-provider consoles are not needed in this course. The cloud labs use LocalStack, fake-gcs-server, and Azurite to exercise selected AWS, Google Cloud Storage, and Azure Blob APIs locally. These are API simulators, not production cloud accounts or substitutes for provider IAM, networking, billing, or availability behavior.

---
