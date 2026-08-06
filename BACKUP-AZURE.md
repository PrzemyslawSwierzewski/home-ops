# Backup PVC na Azure Blob Storage

## Pytanie: Czy to drogo?

**Odpowiedź: Nie, jeśli nie robisz każdego dnia full backup 10 GB.**

### Wycena (2026, region West Europe)
- **Magazynowanie**: ~$0.015 per GB/miesiąc (Hot tier)
- **Transakcje**: ~$0.001 per 10k operacji
- **Egzekuując backup**: $0.006 per GB transferu OUT

**Przykład dla twojego setupu** (16 GB PVC):
- Full backup raz w miesiącu = 1 × 16 GB = $0.10 (transfer) + $0.24 (storage/miesiąc)
- **Total ~$3-4 USD/miesiąc** dla pełnego pokrycia - drażliwie tanie

Jeszcze taniej niż litr drażnicy na Raspberry :)

---

## Setup Azure Storage Account

### 1. Utwórz Storage Account

```bash
az group create --name home-ops-rg --location westeurope

az storage account create \
  --resource-group home-ops-rg \
  --name homeopsbackup$(openssl rand -hex 4) \
  --location westeurope \
  --sku Standard_LRS \
  --access-tier Hot
```

### 2. Utwórz kontainer (bucket)

```bash
STORAGE_ACCOUNT_NAME="homeopsbackup..."  # z kroku wyżej

az storage container create \
  --name pvc-backups \
  --account-name $STORAGE_ACCOUNT_NAME
```

### 3. Wygeneruj klucz dostępu

```bash
az storage account keys list \
  --resource-group home-ops-rg \
  --account-name $STORAGE_ACCOUNT_NAME \
  --query '[0].value' -o tsv
```

**Zapamiętaj**: `Storage Account Name` + `Key` (20 sekund pracy)

---

## Setup Restic + Flux CronJob

### 1. Utwórz Secret w klastrze

```bash
kubectl create secret generic azure-backup \
  --from-literal=AZURE_ACCOUNT_NAME="homeopsbackup..." \
  --from-literal=AZURE_ACCOUNT_KEY="..." \
  -n backup \
  --dry-run=client -o yaml > apps/backup-secret.yaml

# Opcjonalnie: zaszyfruj SOPS
sops -e apps/backup-secret.yaml > apps/backup-secret.enc.yaml
mv apps/backup-secret.enc.yaml apps/backup-secret.yaml
```

### 2. CronJob będzie:

- Tworzyć snapshot każdego PVC
- Spakować restic'iem
- Wysyłać na Azure
- Czyszczć snapshoty

**Harmonogram**:
- **Pełny backup**: 3 AM co tydzień (niedziela)
- **Inkrementalny backup**: Codziennie o 2 AM
- **Retencja**: Ostatnie 4 pełne backupy + 30 dni inkrementów

---

## Przywracanie

Jeśli karta SD padnie:

### Opcja 1: Boot z SSD + Restore

```bash
# 1. Czysty Raspberry Pi OS z SSD
# 2. Zainstaluj k3s (pamiętaj datadir na SSD!)
# 3. Zainstaluj Flux (ze Secret SOPS age)

# 4. Przywróć backupy
restic -r azure:pvc-backups:/ \
  restore latest --target /tmp/restore \
  --include-from-file=/tmp/restore-list.txt

# 5. Przywróć PVC z wygenerowanych plików
kubectl exec -it <pvc-restorer-pod> -- bash
  restic restore <snapshot-id> --target /mnt/pvc
```

### Opcja 2: Azure + Flux (rekomendowane)

Jeśli `prune: true` i Flux zsynchronizuje się ze stanem repo:
- Wszystkie Deploymenty, PVC deklaracje, Secrets (SOPS-encrypted) wrócą
- Restic CronJob automatycznie przywróci dane do PVC

---

## Testowanie

```bash
# 1. Symuluj failure:
rm -rf /var/lib/rancher/k3s/agent/kubelet/pods/  # ← TO NIE RÓB NA PRODUKCJI

# 2. Sprawdź czy backup jest w Azure:
az storage blob list \
  --container-name pvc-backups \
  --account-name $STORAGE_ACCOUNT_NAME

# 3. Restore do test-pod
restic list snapshots ...
restic restore <id> ...
```

---

## Monitoring

Dodaj do `apps/homepage-configmap.yaml`:

```yaml
- name: Backup Status
  widget: custom-api
  url: https://homeopsbackup...blob.core.windows.net/pvc-backups
  headers:
    Authorization: Bearer <SAS token>
```

Lub użyj narzędzia Azure Monitor (free tier + $0.013 per alert).

---

## Koszty przy awarii

| Scenariusz | Koszt | Opis |
|-----------|------|------|
| Kartę SD wymienisz za $10 | $10 | Bez backupu = strata danych |
| SSD 256GB | $20-30 | Boot z SSD zmniejsza wear na SD |
| Azure 1 miesiąc (16 GB) | $4 | Ubezpieczenie przed utratą |
| **Total setup** | ~$40 | **Jednorazowo** |
| **Bieżące koszty** | $4-5/miesiąc | **Tańsze niż kawa** |

---

## Sprawdzenie listy do konfiguracji

- [ ] Storage Account utworzony w Azure
- [ ] Kontener PVC-backups istnieje
- [ ] Klucz storage account zapisany bezpiecznie
- [ ] Secret `azure-backup` w namespace `backup`
- [ ] CronJob `restic-backup-*` zaplanowany
- [ ] Tester restore wykonany (bez faktycznego restoru)
- [ ] Monitoring skonfigurowany
- [ ] SSD podpięty do Raspberry Pi
- [ ] `/etc/fstab` ma `nofail` na starych PVC (na wypadek)

