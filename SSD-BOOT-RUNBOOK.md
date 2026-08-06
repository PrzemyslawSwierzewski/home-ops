# Raspberry Pi 5 - Boot z SSD + Restore Runbook

## Dlaczego SSD?

- **Wear-out**: Karta SD padnie w 12-24 miesiące pod constant write (HA recorder, logs, PVC)
- **Speed**: SSD ~450 MB/s vs SD ~30-40 MB/s → startup 5x szybszy
- **Reliability**: MLC NAND vs TLC na tanich SD → 10x lepsza retencja
- **Cost**: $20-30 za 256 GB SSD << $150 za "custom datacenter recovery"

Na Raspberry Pi 5 możesz boot z USB bez EEPROM patcha (v1.0+).

---

## Hardware

- **SSD**: Samsung 870 QVO 256GB lub Kingston A2000 (liczy się interface - litery po SKU)
- **Adapter**: UASP SATA USB 3.0 adapter (~$10)
- **Kabel**: Zasilacz USB 3.0 5V z dedykowanym prądem (np. z dysku twardego)

⚠️ **UWAGA**: Zasilacz Raspberry Pi musi być 27W+ (z SSD), nie 15W dla samego Pi.

---

## Faza 1: Fizyczna Instalacja

### 1. Przygotuj SSD

```bash
# Na komputerze z Linux (lub Mac z Linux VM):
lsblk
# Powinna być /dev/sdX (np. /dev/sdb - ZMIEŃ NA RZECZYWISTY!)

# Sprawdź czy to SSD
sudo smartctl -i /dev/sdb

# Partycjonuj (MSDOS boot sector OK dla Pi)
sudo parted /dev/sdb \
  mklabel msdos \
  mkpart primary ext4 1MiB 100% \
  align-check opt 1

# Format
sudo mkfs.ext4 -L "rpi-ssd" /dev/sdb1

# Zamontuj
mkdir ~/mnt-ssd
sudo mount /dev/sdb1 ~/mnt-ssd
```

### 2. Przygotuj Raspberry Pi OS

Pobierz Raspberry Pi Imager: https://www.raspberrypi.com/software/

```bash
# W interfejsie wybierz:
# - OS: Raspberry Pi OS (Lite, arm64)
# - Storage: [wybierz SSD przez adapter USB]
# - Advanced: 
#   - Set hostname: rpi-home-ops
#   - Enable SSH: ON
#   - Set username: ops
#   - Set password: <strong-pwd>
#   - Configure wireless LAN: <twoja sieć>
```

### 3. Boot z SSD

```bash
# Na Raspberry Pi podłącz SSD przez USB 3.0 (niebieski port)
# Zasilacz do Pi
# Monitor + keyboard
# Timeout jądra powinien znaleźć SSD ~5 sekund

# Sprawdź czy boot się powiódł:
lsblk
# /dev/sda powinno być SSD

df -h
# / powinno być na /dev/sda1 z ~200+ GB dostępnych
```

---

## Faza 2: Zainstaluj Kubernetes (k3s) na SSD

### 1. k3s z datadir na SSD

```bash
# SSH do Pi
ssh ops@rpi-home-ops

# k3s Lite install na SSD (nie na /var - domyślnie by tam było)
curl -sfL https://get.k3s.io | \
  K3S_DATADIR=/mnt/k3s \
  sh -

# Czekaj aż będzie gotowe
kubectl get nodes -w
```

### 2. Flux + SOPS

```bash
# Zainstaluj Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Bootstrap Flux (wskaż swój repo)
flux bootstrap github \
  --owner=PrzemyslawSwierzewski \
  --repo=home-ops \
  --path=./apps \
  --personal \
  --private=false  # jeśli public repo

# Sprawdzić czy Flux się zaciągnął
kubectl get pods -n flux-system -w
```

### 3. Sekrety SOPS

```bash
# Wyeksportuj klucz age z lokalnego komputera
export AGE_KEY=$(cat ~/.age/keys.txt | grep "AGE-SECRET-KEY" | cut -d: -f2 | xargs)

# Wstrzyknij do clastera
kubectl create secret generic sops-age \
  --from-literal=age.agekey="$AGE_KEY" \
  -n flux-system \
  --dry-run=client -o yaml | kubectl apply -f -

# Flux automatycznie pozwala kryptować sekrety
# Teraz możesz mieć pihole-secret.yaml w repo (encrypted)
```

---

## Faza 3: Aplikuj Manifesty

```bash
# Flux wysłucha repo, ale możesz przyspieszyć:
flux reconcile source git flux-system

# Sprawdzaj reconciliation
flux get kustomizations -w
```

**Wszystkie pody powinny być Running** w ~10 minut.

---

## Faza 4: Restore z Backupu (Po awarii)

### Scenariusz: Karta SD padła

**Masz**: SSD (boot), Azure Blob (backupy), repo GitHub (manifesty)

#### Krok 1: Uniwersalne odzyskiwanie (5 minut)

```bash
# Boot z SSD (już działa)
# Flux natychmiast reconcile repo → wszystkie PVC wrócą (puste!)

# Ale dane w PVC są w Azure:
restic -r azure:pvc-backups:/ snapshots
```

#### Krok 2: Restore PVC

```bash
# Opcion A: Direct restore (szybko, ale dużo IO)
# Uruchom pod z restic
kubectl run -it --rm restic-restore \
  --image=restic/restic:latest \
  --serviceaccount=restic-backup \
  -n backup \
  -- bash

# W pod'zie:
export AZURE_ACCOUNT_NAME=...  # z secretu
export AZURE_ACCOUNT_KEY=...
export RESTIC_REPOSITORY=azure:pvc-backups:/
export RESTIC_PASSWORD=...

restic snapshots  # Pokaż dostępne snapshoty

# Restore najnowszego snapshotę do /tmp
restic restore latest --target /tmp
```

#### Krok 3: Skopiuj dane z /tmp do montowanego PVC

```bash
# W tym samym pod'zie:
kubectl exec -it <homeassistant-pod> -n home-automation -- bash

# Z hosta:
kubectl cp <restic-restore-pod>:/tmp/backup/homeassistant/* \
  home-automation/<homeassistant-pod>:/config/ \
  -n backup

# Dla HA restart:
kubectl rollout restart deployment/homeassistant -n home-automation
```

#### Opcja B: Pełny Snapshot Restore (zalecane)

Jeśli masz dostęp do Azure CLI na Pi:

```bash
# Pobierz najnowszy snapshot z metadata
az storage blob list \
  --container-name pvc-backups \
  --account-name $AZURE_ACCOUNT_NAME \
  --query 'sort_by(@, &properties.lastModified)[-1]'

# Utwórz nowy PVC z tymi danymi
# (zamiast restore - rebuild z backup jako PV)
```

---

## Faza 5: Monitoring - Czy SSD żyje?

```bash
# Codziennie sprawdzaj SMART:
sudo smartctl -H /dev/sda

# Temperaturę
cat /sys/class/thermal/thermal_zone*/temp

# Wear level
sudo smartctl -A /dev/sda | grep -E "NAND|Wear"
```

Jeśli **Wear_Leveling_Count < 5%** lub **Reallocated_Sector_Ct > 10**, zastąp SSD.

---

## Checklist Implementacji

- [ ] SSD 256GB+ zakupiony
- [ ] Adapter SATA→USB 3.0 zakupiony + zasilacz 27W+
- [ ] Raspberry Pi OS (arm64, Lite) na SSD
- [ ] SSH na Pi działa
- [ ] k3s zainstalowany z `/mnt/k3s` jako datadir
- [ ] Flux bootstrapped do GitHub repo
- [ ] SOPS age-key zainstalowany w flux-system
- [ ] Manifesty z Azure backup secret (encrypted) w repo
- [ ] CronJob restic-backup-full zaplanowany
- [ ] Restic init na Azure Blob Storage
- [ ] Test restore wykonany (dry-run)
- [ ] Monitoring health SSD skonfigurowany

---

## Timeframe

| Etap | Czas |
|------|------|
| 1. Fizyczna instalacja | 15 min |
| 2. Flux + k3s setup | 20 min |
| 3. Manifesty | 5 min |
| 4. Pierwszy backup | 30 min |
| 5. Test restore | 15 min |
| **TOTAL** | **1.5 godziny** |

---

## Backup Plan dla Backup Planu

Jeśli Azure padnie (rzadko) lub potrzebujesz offline copy:

```bash
# Cyklicz: raz na miesiąc na lokalnym HDD
restic -r /mnt/backup-external-hdd dump latest / > /mnt/monthly-backup.tar

# Przechowuj: W innym miejscu (dom kolegi / pendrive w sejfie)
```

**Koszt**: 0 (pendrive możesz mieć)  
**Zysk**: Offline disaster recovery vs "uszkodzenie Azure" scenario

