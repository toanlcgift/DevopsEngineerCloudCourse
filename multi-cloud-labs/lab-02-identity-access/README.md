# Lab 02 – Identity and Access Management

Explore principals, roles, permissions, and least privilege using a local policy decision point. Azure RBAC and Google Cloud IAM are managed control planes and do not have complete local emulators; this lab does not claim to provision real identities. Complete the [WSL setup](../README.md#start-here).

## 1. Install Open Policy Agent (OPA)

OPA evaluates policy locally without a cloud account:

```bash
echo "Step 1/2: Download the OPA CLI for Linux x86_64"
curl -fsSLo /tmp/opa https://openpolicyagent.org/downloads/latest/opa_linux_amd64_static
sudo install -m 0755 /tmp/opa /usr/local/bin/opa
rm /tmp/opa
echo "Step 2/2: Create this lab's policy workspace"
mkdir -p ~/devops-labs/lab02
cd ~/devops-labs/lab02
```

## 2. Write a least-privilege policy and test cases

The policy permits a reader to list and read objects in one named bucket. It denies writes and access to other buckets.

```bash
cat > policy.rego <<'EOF'
package lab.iam

default allow := false

allow if {
  input.principal.role == "reader"
  input.action == "list"
  input.resource.bucket == "course-artifacts"
}

allow if {
  input.principal.role == "reader"
  input.action == "get"
  input.resource.bucket == "course-artifacts"
}
EOF

cat > cases.json <<'EOF'
[
  {"principal":{"id":"alice","role":"reader"},"action":"get","resource":{"bucket":"course-artifacts"},"expected":true},
  {"principal":{"id":"alice","role":"reader"},"action":"list","resource":{"bucket":"course-artifacts"},"expected":true},
  {"principal":{"id":"alice","role":"reader"},"action":"put","resource":{"bucket":"course-artifacts"},"expected":false},
  {"principal":{"id":"alice","role":"reader"},"action":"get","resource":{"bucket":"payroll"},"expected":false}
]
EOF
```

## 3. Evaluate allow and deny cases

```bash
opa eval --format pretty --data policy.rego \
  --input <(jq -c '.[0]' cases.json) 'data.lab.iam.allow'
opa eval --format pretty --data policy.rego \
  --input <(jq -c '.[2]' cases.json) 'data.lab.iam.allow'
```

`opa test` reports no tests yet unless test rules are added. Add explicit tests and run them:

```bash
cat > policy_test.rego <<'EOF'
package lab.iam

test_reader_can_get if {
  allow with input as {"principal":{"role":"reader"},"action":"get","resource":{"bucket":"course-artifacts"}}
}

test_reader_cannot_put if {
  not allow with input as {"principal":{"role":"reader"},"action":"put","resource":{"bucket":"course-artifacts"}}
}
EOF
opa test . --verbose
```

## 4. Extend the exercise

Add a `writer` role that can write only to `course-artifacts`, and tests proving it cannot access `payroll`. Discuss how AWS IAM policies, Azure role assignments/scopes, and Google Cloud IAM bindings express similar intent differently. In real cloud accounts, confirm the principal, scope, inherited grants, and deny behavior using provider documentation; never attach broad administrator roles just to make a lab pass.

```bash
opa fmt --write policy.rego policy_test.rego
opa test . --verbose
```

No containers or cloud resources were created by this lab.
