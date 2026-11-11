# Lab 14 – Configuration Management with Ansible

Install Ansible, manage localhost using an inventory, and run an idempotent playbook that creates a small web document. No remote hosts or cloud resources are needed. Complete the [WSL setup](../README.md#start-here).

## 1. Install Ansible and write inventory/playbook

```bash
sudo apt-get update
sudo apt-get install -y ansible
mkdir -p ~/devops-labs/lab14
cd ~/devops-labs/lab14
cat > inventory.ini <<'EOF'
[local]
localhost ansible_connection=local
EOF

cat > site.yml <<'EOF'
---
- name: Configure a local lab document
  hosts: local
  gather_facts: false
  vars:
    lab_dir: "{{ lookup('env', 'HOME') }}/devops-labs/ansible-target"
  tasks:
    - name: Create the application directory
      ansible.builtin.file:
        path: "{{ lab_dir }}"
        state: directory
        mode: "0750"
    - name: Install the index document
      ansible.builtin.copy:
        dest: "{{ lab_dir }}/index.html"
        content: "<h1>Configured by Ansible</h1>\n"
        mode: "0640"
EOF
```

## 2. Check and run the playbook

```bash
ansible --version
ansible-inventory -i inventory.ini --list
ansible-playbook -i inventory.ini site.yml --syntax-check
ansible-playbook -i inventory.ini site.yml --check --diff
ansible-playbook -i inventory.ini site.yml
cat ~/devops-labs/ansible-target/index.html
ansible-playbook -i inventory.ini site.yml
```

The second successful run should report no changes, demonstrating idempotence. For remote hosts, use SSH keys and narrowly scoped sudo; do not put passwords or private keys in inventory files. Keep `--check --diff` in your review workflow but remember check mode cannot predict every module effect.

## 3. Clean up

```bash
rm -rf ~/devops-labs/lab14 ~/devops-labs/ansible-target
```
