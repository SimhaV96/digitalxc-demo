# k3s Hello World CI/CD Challenge

## Prerequisites
- EC2 instance: Ubuntu 24.04, t3.medium, running
- Security Group: allow inbound 22 (SSH) and 80 (HTTP) from your IP
- SSH access to the instance

## Part 1: Install k3s with Ansible (run locally on the EC2 instance)

1. SSH into the instance and install Ansible:
   ```bash
   ssh -i ~/.ssh/your-key.pem ubuntu@YOUR_EC2_PUBLIC_IP
   sudo apt update
   sudo apt install -y software-properties-common
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt install -y ansible
   ```

2. Clone this repo onto the instance:
   ```bash
   git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git ~/k3s-challenge
   cd ~/k3s-challenge
   ```

3. `ansible/inventory.ini` is set up to run locally (Ansible executes directly on this same machine, no separate control node):
   ```ini
   [k3s_node]
   localhost ansible_connection=local
   ```

4. Run the playbook:
   ```bash
   ansible-playbook -i ansible/inventory.ini ansible/install-k3s.yml --ask-become-pass
   ```
   This installs the latest stable k3s as a single-node cluster, **with Traefik disabled** (`--disable traefik`) — we don't need an ingress controller for this challenge, and leaving it enabled causes a port-80 conflict with our own LoadBalancer service later.

5. Verify:
   ```bash
   kubectl get nodes
   kubectl get pods -A
   ```
   One node in `Ready` status; only `coredns`, `local-path-provisioner`, and `metrics-server` in `kube-system` (no traefik pods).

## Part 2: Kubernetes Manifests

Manifests are in `k8s/`:
- `configmap.yaml` — the static Hello World HTML page
- `deployment.yaml` — Nginx deployment mounting that page
- `service.yaml` — `LoadBalancer` service exposing it on port 80 via k3s's built-in ServiceLB

Test manually first, from the EC2 instance:
```bash
kubectl apply -f k8s/configmap.yaml
kubectl apply -f k8s/deployment.yaml
kubectl apply -f k8s/service.yaml
kubectl get pods
kubectl get svc hello-world-nginx-service
```
`EXTERNAL-IP` should populate with the node's IP. Then in a browser: `http://YOUR_EC2_PUBLIC_IP` — should show "Hello World".

## Part 3: GitHub Actions Self-Hosted Runner + Pipeline

**Why self-hosted runner instead of a kubeconfig secret:** k3s's API server isn't exposed to the internet — the eval criteria explicitly calls out secure secret handling. A self-hosted runner living on the same EC2 instance reaches the local k3s API directly with zero exposed ports and zero secrets to manage.

1. On GitHub: repo → Settings → Actions → Runners → "New self-hosted runner" → Linux. Run the generated commands on your EC2 instance one at a time (avoid pasting the whole block — GitHub's UI includes "Copied!" button text that can get captured):
   ```bash
   mkdir actions-runner && cd actions-runner
   curl -o actions-runner-linux-x64.tar.gz -L <URL_FROM_GITHUB>
   tar xzf actions-runner-linux-x64.tar.gz
   ./config.sh --url https://github.com/YOUR_USERNAME/YOUR_REPO --token YOUR_TOKEN
   ```

2. Install as a persistent background service (don't use `./run.sh` directly — that only runs in the foreground and stops when you disconnect):
   ```bash
   sudo ./svc.sh install
   sudo ./svc.sh start
   sudo ./svc.sh status
   ```
   Confirm on GitHub: repo → Settings → Actions → Runners — should show **Idle** (green dot).

3. Any push touching `k8s/**` on `main` triggers `.github/workflows/deploy.yml`, which:
   - Applies the ConfigMap, Deployment, and Service manifests
   - Forces a rollout restart so ConfigMap changes are picked up immediately, rather than waiting on the kubelet's periodic sync interval (~60s) for mounted volume updates
   - Waits for the rollout to complete and prints final pod/service status

4. Watch it run: GitHub repo → Actions tab.

5. Confirm in browser: `http://YOUR_EC2_PUBLIC_IP`

## Recording Checklist
- [ ] Run `ansible-playbook`, show it complete successfully
- [ ] `kubectl get nodes` and `kubectl get pods -A` showing Ready, no Traefik
- [ ] `kubectl get svc` showing `EXTERNAL-IP` populated on port 80
- [ ] Push a small change (e.g. edit the HTML text in `configmap.yaml`) to `main`
- [ ] Show GitHub Actions tab — pipeline triggers and runs automatically, including the rollout restart step
- [ ] Refresh browser at `http://YOUR_EC2_PUBLIC_IP` — show the updated content live

## Notes / Design Decisions
- **Ansible over Terraform**: Terraform provisions infrastructure (the EC2 instance itself); Ansible configures software on an already-running machine. Since the instance already exists, this is a configuration task — Ansible's actual purpose.
- **Traefik disabled**: k3s bundles Traefik and ServiceLB by default. Traefik's own LoadBalancer claims port 80 on the node; our app's LoadBalancer service needs that same port, so Traefik is disabled at install time to avoid the conflict.
- **LoadBalancer over NodePort**: NodePort services are restricted to ports 30000–32767 by Kubernetes validation, so port 80 isn't possible with `type: NodePort`. k3s's built-in ServiceLB supports `type: LoadBalancer` on bare-metal/EC2 without needing a cloud provider's native LB, and isn't subject to the NodePort range restriction.
- **Self-hosted runner over kubeconfig secret**: avoids exposing the k3s API server publicly and avoids storing cluster credentials as a GitHub secret entirely — the runner already has local access to the cluster.
