# Tomorrow — Portfolio Task #2

## Context
We're starting **ARM64 Build** of `dirtie-srv` for the Raspberry Pi k3s cluster. If build goes smoothly, we'll likely knock out k8s manifests and possibly deploy.

---

## 1. ARM64 Build `[COLLAB]`
**Goal:** Patch `dirtie-srv/Dockerfile` to produce a `linux/arm64` image and confirm it runs on a Pi.

### What's broken now
```dockerfile
FROM golang:latest              # amd64 by default on build host
RUN GOOS=linux go build ...    # missing GOARCH=arm64
```

### Fix options
- **(A) Multi-stage ARM64 bases** (recommended)
  ```dockerfile
  FROM --platform=linux/arm64 golang:1.23-bookworm AS builder
  ...
  RUN CGO_ENABLED=0 GOOS=linux GOARCH=arm64 go build ./cmd/main.go
  FROM --platform=linux/arm64 debian:bookworm-slim
  ```
- **(B) Buildx cross-compile** from amd64 host:
  ```bash
  docker buildx build --platform linux/arm64 -t dirtie-srv:arm64 .
  ```

### Done when
- [ ] Image builds without error
- [ ] `docker inspect` shows `Architecture: arm64`
- [ ] Pushed to registry (or `docker save` + `scp` to Pi)
- [ ] Runs on target Pi: `docker run --rm <image> /app/main --help` succeeds

---

## 2. k8s Manifests `[AI_AGENT]`
**Goal:** Create `~/projects/k3setup/kubernetes/apps/dirtie-srv/` with basics.

### Files
| File | Purpose |
|------|---------|
| `deployment.yaml` | ARM64 image, containerPort 8000, `envFrom` secret |
| `service.yaml` | ClusterIP on port 8000 |
| `kustomization.yaml` | Base kustomize layer |
| `namespace.yaml` | `dirtie` namespace |

### Key decisions needed
- Image registry: Docker Hub? GitHub Container Registry? Local registry?
- Secrets: `.env` from repo → `kubectl create secret generic` or SealedSecret/ExternalSecret?
- Resource requests: start small (`10m` CPU, `64Mi` mem) for Pi limits

### Done when
- [ ] `kubectl apply -k .` dry-run validates
- [ ] No hardcoded credentials in YAML

---

## 3. Apply Manifests `[USER]` *(if time)*
**Goal:** Deploy to k3s control plane.

### Steps
1. SSH into control plane Pi
2. `scp` or `git clone` manifest dir to Pi
3. `kubectl apply -k apps/dirtie-srv/`
4. `kubectl get pods -n dirtie -w`

### Traps to watch for
- `ImagePullBackOff` → registry auth missing or wrong `imagePullPolicy`
- `CrashLoopBackOff` → `.env` vars not injected; check container logs
- `Pending` → taint/toleration mismatch if node had etcd/control-plane role

### Done when
- [ ] Pod reports `Running`
- [ ] Port-forward or NodePort/ingress reaches `http://<pi>:8000/health`

---

## 4. State Decision `[USER]` *(if we get this far)*
**Goal:** Decide where InfluxDB / Postgres / Mosquitto / Grafana live.

### Quick pros/cons
| Approach | Pros | Cons |
|----------|------|------|
| **On-cluster** (`hostPath` / `local-path-provisioner` / StatefulSet) | Single kubectl commands, everything in one place | SD card wear, no HA on 3-node, Pi I/O bound |
| **Off-cluster** (docker-compose on durable Pi / NAS) | Isolated I/O, easier backups, protects cluster | Two sources of truth, need to manage firewall/Tailscale |

**Bias:** For 3 SD-card Pis, **off-cluster stateful stack** is usually more reliable. Keep dirtie-srv + Traefik on k3s; move DB/broker to a compose file on the most reliable node.

### Done when
- [ ] Decision recorded in `k3setup/docs/state_decision.md`
- [ ] Next task (write manifests or compose) is unblocked

---

## Rough order
1. You + me → patch Dockerfile, build ARM64 image, test on Pi
2. Me → write k8s manifests while you verify image on hardware
3. You → apply manifests (SSH, kubectl)
4. Pair → debug live if pods don't come up
5. You → make State Decision if we have bandwidth left

---

## Useful commands to have open
```bash
# ARM64 build from amd64 host
docker buildx build --platform linux/arm64 -t dirtie-srv:arm64 .

# Inspect image arch
docker inspect --format='{{.Os}}/{{.Architecture}}' dirtie-srv:arm64

# Save + copy to Pi
docker save dirtie-srv:arm64 | ssh pi@192.168.0.XXX 'docker load'

# Check k3s on Pi
ssh pi@<ip> 'sudo systemctl status k3s'
```
