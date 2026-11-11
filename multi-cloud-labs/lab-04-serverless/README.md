# Lab 04 – Serverless Application Development

Build and run an HTTP-triggered Python function locally with the Functions Framework. This verifies handler behavior without creating a cloud function. The framework is not a full emulator of AWS Lambda, Azure Functions, or Google Cloud Functions. Complete the [WSL setup](../README.md#start-here).

## 1. Create a Python project

```bash
mkdir -p ~/devops-labs/lab04
cd ~/devops-labs/lab04
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install functions-framework
```

## 2. Write an HTTP function

```bash
cat > main.py <<'EOF'
import json
import os


def hello(request):
    name = request.args.get("name", "student")
    response = {
        "message": f"Hello, {name}!",
        "environment": os.getenv("APP_ENV", "development"),
    }
    return (json.dumps(response), 200, {"Content-Type": "application/json"})
EOF

cat > requirements.txt <<'EOF'
functions-framework
EOF
```

## 3. Run and test the HTTP endpoint

```bash
source .venv/bin/activate
export APP_ENV=lab
functions-framework --target=hello --host=127.0.0.1 --port=8080
```

Keep that terminal open. In a second WSL terminal, send requests:

```bash
curl --fail-with-body 'http://127.0.0.1:8080/?name=Ubuntu'
curl --fail-with-body 'http://127.0.0.1:8080/'
```

Stop the server with `Ctrl-C`. Add validation for missing or overlong names and return a suitable error status. For deployment, use each provider's supported Python runtime and identity mechanism; local HTTP success does not test cloud IAM, quotas, cold starts, or billing.

## 4. Clean up

```bash
rm -rf ~/devops-labs/lab04
```
