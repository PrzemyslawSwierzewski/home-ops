# Home Ops – Kubernetes Home Automation

Fully automated, GitOps-managed home automation cluster running on **Raspberry Pi 5 (8GB RAM)** with **k3s**.

## 🏗️ Architecture

```
Raspberry Pi 5 (8GB)
└── k3s (lightweight Kubernetes)
    ├── Flux CD (GitOps reconciliation)
    ├── Home Assistant (home automation hub)
    ├── Zigbee2MQTT (Zigbee device bridge)
    ├── Mosquitto (MQTT broker)
    ├── Pi-hole (DNS/ad blocker)
    ├── Homepage (dashboard)
    ├── Stirling-PDF (PDF tools)
    └── Tailscale (secure VPN access)
```

## 📋 Services

| Service | Port | Purpose | Storage |
|---------|------|---------|---------|
| **Home Assistant** | 8123 | Home automation hub | 10 GiB PVC |
| **Zigbee2MQTT** | 8080 | Zigbee device coordinator | 2 GiB PVC |
| **Mosquitto** | 1883 | MQTT message broker | 2 GiB PVC |
| **Pi-hole** | 80, 53 (DNS) | Ad blocking & DNS | 2 GiB PVC |
| **Homepage** | 3000 | Web dashboard | ConfigMap |
| **Stirling-PDF** | 8080 | PDF manipulation | None |

All services exposed via **Tailscale Ingress** (no port forwarding, secure by default).

**DNS**: Configured at Tailscale network level to query Pi-hole directly for all home-automation queries and for all devices in the tailscale network queries.

## 🚀 Quick Start

### Prerequisites

- **Raspberry Pi 5** (8GB RAM minimum)
- **Zigbee coordinator** (Sonoff Zigbee 3.0 USB Dongle Plus V2, or compatible)
- **k3s** already running on Pi

### Installation

1. **Install Flux CD** (if not already done)

```bash
curl -s https://fluxcd.io/install.sh | sudo bash
flux bootstrap github \
  --owner=PrzemyslawSwierzewski \
  --repo=home-ops \
  --branch=main \
  --path=./apps
```

2. **Wait for GitOps sync**

```bash
flux get kustomization --watch
# All kustomizations should show READY=True

kubectl get pods -A
# All pods in home-automation, pihole, homepage, stirling-pdf should be Running
```

3. **Access services via Tailscale**

All services are accessible only within your Tailscale network:
- Home Assistant: `https://homeassistant.tail...ts.net`
- Pi-hole Admin: `https://pihole.tail...ts.net/admin`
- Zigbee2MQTT: `https://home-automation-zigbee2mqtt.tail...ts.net/`
- Homepage Dashboard: `https://homepage.tail...ts.net`
- Stirling-PDF: `https://stirling-pdf.tail...ts.net`

## 🔧 Kubernetes Configuration

### Health Probes

All stateful services have:
- **startupProbe**: Waits for service initialization (HA takes 2-3 min on first start)
- **readinessProbe**: Checks if pod can handle traffic
- **livenessProbe**: Restarts pod if unhealthy (e.g., connection lost)

Kubelet automatically restarts failed pods — no manual intervention needed.

**Example**: If Z2M loses connection to Zigbee coordinator:
1. readinessProbe fails → Pod marked "Not Ready" (Ingress stops routing to it)
2. livenessProbe reaches failureThreshold → Pod restarts automatically
3. New pod starts, reconnects to coordinator

### Resource Management

| Service | CPU Request | CPU Limit | Memory Request | Memory Limit |
|---------|------------|-----------|----------------|-------------|
| HA | 200m | 800m | 512Mi | 1536Mi |
| Z2M | 50m | 500m | 128Mi | 512Mi |
| Mosquitto | 10m | 200m | 32Mi | 128Mi |
| Pi-hole | 50m | 200m | 128Mi | 512Mi |
| Stirling-PDF | 100m | 800m | 512Mi | 2Gi |

**Priority Classes**:
- `home-critical`: Pi-hole (DNS for entire home network)
- `home-important`: HA, Z2M, Mosquitto (critical for automation)
- `home-default`: Homepage, Stirling-PDF (non-critical UI)

On memory pressure, kubelet kills lower-priority pods first to keep DNS and automation running.

### Deployment Strategy

All services use **Recreate** strategy (not RollingUpdate):
- **Why**: PVCs are `ReadWriteOnce` (can only be mounted on one node at a time)
- **How**: Old pod is killed → volume unmounts → new pod starts → volume remounts
- Prevents mount deadlock that would occur with RollingUpdate

**Startup times**:
- HA: ~3 min (includes database migration on updates)
- Z2M: ~30 sec (waits for Zigbee coordinator detection)
- Mosquitto: ~5 sec
- Pi-hole: ~10 sec

## 📁 Project Structure

```
home-ops/
├── .renovaterc.json           # Auto-update image tags (weekly PR)
├── .gitignore                 # Excludes secrets (currently unused)
├── .sops.yaml                 # SOPS encryption config (currently unused)
├── README.md                  # This file
├── BACKUP-AZURE.md            # Azure backup setup (currently unused, I'm just downloading backups on my PC)
├── apps/                      # Kubernetes manifests (applied by Flux)
│   ├── kustomization.yaml     # Kustomize build root
│   ├── priority-classes.yaml  # Pod priority definitions
│   ├── config/                # Homepage configuration (separate files)
│   │   ├── settings.yaml      # Dashboard title, theme, background
│   │   ├── services.yaml      # Service cards and links
│   │   ├── bookmarks.yaml     # Quick links
│   │   ├── widgets.yaml       # Resource monitors
│   │   └── custom.css         # Styling
│   ├── flux-system/           # Flux CD configuration
│   │   ├── gotk-components.yaml
│   │   └── gotk-sync.yaml
│   ├── *-deployment.yaml      # Service deployments (HA, Z2M, etc.)
│   ├── *-service.yaml         # Service definitions
│   ├── *-ingress.yaml         # Tailscale ingress rules
│   ├── *-PVC.yaml             # Persistent volumes (storage)
│   ├── *-configmap.yaml       # Service configurations
│   ├── *-namespace.yaml       # Kubernetes namespaces
│   └── tailscale-operator.yaml # Tailscale operator config
└── .git/                      # Git repository history
```

## 🔄 GitOps Workflow

### Automatic Reconciliation

Flux CD continuously reconciles the cluster state with the git repository:

1. **Every 1 minute**: Flux checks for new commits in `main` branch
2. **On change**: Automatically applies manifests from `./apps`
3. **On error**: Retries until successful, logs errors in kustomization status

**Check status**:
```bash
flux get kustomization
# Should show READY=True and recent REVISION
```

**Manual sync** (force immediate reconciliation):
```bash
flux reconcile kustomization flux-system
```

### Image Updates with Renovate

Every week, Renovate automatically:
1. Checks Docker Hub/GitHub Container Registry for new image tags
2. Creates a PR with updated `image:` fields in manifests
3. Auto-merges non-critical updates (busybox)
4. Requires manual approval for critical services

**Current pinned versions** (checked by Renovate):
- Home Assistant: `2026.8.0`
- Zigbee2MQTT: `2.13.0`
- Mosquitto: `2.1.2`
- Pi-hole: `2026.07.2`
- Homepage: `v1.13.2`
- Stirling-PDF: `2.14.2-ultra-lite`
- busybox: `1.38.0`

## ⚠️ Important Notes

### Zigbee Network Preservation

Zigbee2MQTT's `configuration.yaml` contains critical data:
- **Zigbee network key** (encryption key for all devices)
- **PAN ID** (network identifier)
- **Paired device list** (addresses of all your Zigbee devices)

The init container **only copies config on first start**:
```bash
if [ -f /app/data/configuration.yaml ]; then
  echo "config exists - not overwriting"
else
  echo "first start - copying from ConfigMap"
  cp /tmp/config/configuration.yaml /app/data/configuration.yaml
fi
```

This means:
- Pod restarts preserve the network key ✓
- Device pairing survives restarts ✓
- You can edit the ConfigMap without losing the network ✓

If you need to reset: manually delete the PVC and redeploy.

### Pi-hole as Home DNS

Pi-hole is configured with `home-critical` priority class. On memory pressure:
- Other pods are evicted first
- DNS service stays up
- Home network can always resolve names

**DNS configuration** (at Tailscale level):
- Tailscale routes all `*.home` queries to Pi-hole's private IP
- Other queries go to system DNS

### Zigbee Coordinator Device Mount

The Zigbee coordinator is mounted as `/dev/serial/by-id/usb-Itead_Sonoff_...`:
- **Stable symlink**: survives USB port changes, Pi reboots
- **Automatically discovered**: Z2M finds it via this stable path
- **CharDevice mount**: direct access to serial port

## 🔐 Security & Privacy

- **No port forwarding**: All services only accessible via Tailscale VPN
- **No public internet exposure**: Entire home network isolated
- **VPN-gated access**: Only devices in your Tailscale network can reach services
- **HTTPS/TLS**: Tailscale automatically handles certificate generation, termination, and rotation
  - Each Ingress gets a self-signed cert trusted within the Tailscale network
  - No cert-manager needed — Tailscale manages all certificate lifecycle
  - All connections are encrypted end-to-end
- **Future**: SOPS + age for secret encryption in git repo

## 📊 Health & Monitoring

### Memory Usage at Full Load

Typical memory consumption with all services running:
- Home Assistant: 200-400 MB
- Zigbee2MQTT: 100-150 MB
- Mosquitto: 20-30 MB
- Pi-hole: 100-200 MB
- Stirling-PDF: 100-500 MB (spikes during PDF processing)
- k3s system + kubelet: ~1.5 GB

**Total**: ~2.5-3.0 GB → **Safe** on 8GB Pi with headroom for peaks.


## 📈 Future Improvements

- [ ] **SOPS + age encryption** for secret management
- [ ] **Boot from SSD** (currently using SD card)
- [ ] **NetworkPolicy** for inter-pod communication controls
- [ ] **Prometheus + Grafana** for monitoring and visualization

## 📚 Further Reading

- [Flux CD Documentation](https://fluxcd.io/docs/)
- [k3s Documentation](https://docs.k3s.io/)
- [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Home Assistant Documentation](https://www.home-assistant.io/docs/)
- [Zigbee2MQTT Documentation](https://www.zigbee2mqtt.io/guide/getting-started/)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [Tailscale Documentation](https://tailscale.com/kb/)

## 🔗 Links

- **GitHub Repository**: https://github.com/PrzemyslawSwierzewski/home-ops
- **Home Assistant**: https://www.home-assistant.io/
- **k3s**: https://k3s.io/
- **Flux CD**: https://fluxcd.io/
- **Tailscale**: https://tailscale.com/

---

**Last updated**: August 9, 2026  
**Status**: ✅ Stable, running in production  
**Hardware**: Raspberry Pi 5 (8GB RAM), k3s  
**Owner**: @PrzemyslawSwierzewski  
**License**: MIT