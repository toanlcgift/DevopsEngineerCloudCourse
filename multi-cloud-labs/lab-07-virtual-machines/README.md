# Lab 07 – Virtual Machine Deployment

Practice Linux service administration inside an isolated Ubuntu container. A container is **not** a virtual machine: it shares the host kernel and cannot demonstrate a cloud hypervisor, VM boot process, or full guest networking. WSL is not assumed to support nested virtualization. Complete the [Docker setup](../README.md#start-here).

## 1. Start a disposable Ubuntu guest-like container

```bash
docker run -d --name lab07-ubuntu \
  -p 127.0.0.1:8087:80 \
  ubuntu:24.04 sleep infinity
docker exec lab07-ubuntu apt-get update
docker exec lab07-ubuntu apt-get install -y nginx curl
docker exec lab07-ubuntu sh -c 'printf "%s\n" "<h1>Lab 07 web server</h1>" > /var/www/html/index.html'
docker exec -d lab07-ubuntu nginx
```

## 2. Verify service and connectivity

```bash
curl --fail-with-body http://127.0.0.1:8087/
docker exec lab07-ubuntu nginx -t
docker exec lab07-ubuntu service nginx status
docker inspect lab07-ubuntu --format '{{.HostConfig.PortBindings}}'
```

The port is explicitly bound to loopback. Compare this with a cloud VM's public IP, subnet route, firewall/security-group rules, and SSH identity. Do not expose SSH or a development web service to the public internet.

## 3. Explore logs, restart, and cleanup

```bash
docker exec lab07-ubuntu sh -c 'echo "lab request" >> /var/log/nginx/access.log'
docker exec lab07-ubuntu tail -n 5 /var/log/nginx/access.log
docker restart lab07-ubuntu
docker exec lab07-ubuntu service nginx start
curl --fail-with-body http://127.0.0.1:8087/
docker rm -f lab07-ubuntu
```

For a true VM exercise, use an approved hypervisor or a disposable cloud account and follow its current networking, SSH-key, cost, and teardown guidance. This guide intentionally does not create one.
