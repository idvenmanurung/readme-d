# Panduan Demo — To The Point (Tanpa Analogi)

---

## 1. Project ini buat apa?

Automation untuk deploy **Kubernetes cluster on-premise** pakai **RKE2** (distro Kubernetes) dan **Rancher** (UI + control plane buat manage cluster), yang nantinya dipakai jalanin aplikasi **GLChat/GLoria**.

Alur besarnya: siapin bastion → preflight check → install dependency di semua node → setup load balancer → deploy Rancher → bikin cluster RKE2 → register node master & worker → ambil kubeconfig → pasang add-ons (ingress, storage) → label/taint node → (opsional) setup GPU node.

Task yang kamu kerjain: bikin jalur pintas khusus buat 2 step terakhir (ambil kubeconfig + label/taint), tanpa harus jalanin semua step dari awal. Dipakai untuk kasus cluster sudah ada duluan dan cuma butuh dipasang label/taint.

---

## 2. Kenapa ada 2 `docker-compose` di folder rancher?

| File | Fungsi |
|---|---|
| `infra/rancher/docker-compose-rancher.yml` | Menjalankan container **Rancher server itu sendiri** (image `rancher/rancher:v2.11.3`). Dipakai `setup.sh`, isi variabel `PASSWORD` disuntik pakai `envsubst`, hasilnya di-`docker compose up -d` di server. |
| `infra/rancher/chromedp/docker-compose.yaml` | Menjalankan container automation (`chromedp-rancher-bootstrapper`) yang login otomatis ke Rancher pakai browser headless dan accept EULA — karena Rancher versi baru mewajibkan klik "accept" di UI saat pertama kali install, dan itu tidak bisa di-skip lewat API biasa. |

Dua file beda tujuan, sengaja dipisah folder.

---

## 3. Fungsi Tiap Script yang Relevan

| Script | Fungsi |
|---|---|
| `infra/scripts/preflight-check.sh` | Validasi `config.yml`, cek `yq` terinstall, cek SSH ke semua node bisa connect, sebelum proses install dimulai. |
| `infra/scripts/install-dependencies.sh` | SSH ke tiap server di `config.yml`, install dependency dasar (Docker, RKE2 binary). |
| `infra/scripts/setup-docker.sh` | Install Docker Engine. Dipanggil dari beberapa script lain termasuk `infra/rancher/setup.sh`. |
| `infra/scripts/setup-load-balancer.sh` | Deploy Nginx/HAProxy di depan Rancher sebagai reverse proxy. |
| `infra/rancher/setup.sh` | Orkestrator Rancher: install Docker → jalankan `docker-compose-rancher.yml` → tunggu Rancher API ready → login → trigger EULA automation (chromedp) → panggil `create-cluster.sh`. |
| `infra/rancher/create-cluster.sh` | Kirim request ke Rancher API buat provisioning objek Cluster (RKE2), termasuk setting TLS SAN (list IP/domain yang valid buat akses API server). |
| `infra/rancher/register-all-nodes-worker.sh` | Login ke Rancher API, ambil registration token, SSH ke tiap node untuk jalankan `system-agent-install.sh` supaya node join ke cluster sesuai `role` (etcd/controlplane/worker) di `config.yml`. |
| `infra/scripts/setup-kubeconfig.sh` | SSH ke master node, ambil `/etc/rancher/rke2/rke2.yaml`, ganti IP internal jadi IP load balancer, simpan ke `~/.kube/config`, export `KUBECONFIG`. |
| `infra/scripts/setup-add-ons.sh` | Baca `config.yml` bagian `add-ons`, install Helm chart (ingress-nginx) dan manifest kubectl (local-path-provisioner). |
| `infra/scripts/setup-node-label-taints.sh` | Baca `infra.rke2.nodes[*].labels` dan `.taints` dari `config.yml`, loop tiap node, jalankan `kubectl label` dan `kubectl taint`. |
| `infra/scripts/setup-gpu-node.sh` | Cari node dengan `type: gpu` di config, install NVIDIA driver + toolkit. |

---

## 4. Baris-per-Baris Perubahan di Makefile

```makefile
setup-node-labels-taints:
	@echo "Fetching kubeconfig..."
	bash infra/scripts/setup-kubeconfig.sh
	@echo "Applying node labels and taints from config.yml..."
	bash infra/scripts/setup-node-label-taints.sh
```

| Baris | Fungsi | Kalau baris ini dihapus |
|---|---|---|
| `setup-node-labels-taints:` | Nama target — inilah yang bikin `make setup-node-labels-taints` valid dikenali `make`. | `make` error: `No rule to make target`. |
| `@echo "Fetching kubeconfig..."` | Print status ke terminal. `@` di depan artinya command echo-nya sendiri tidak ikut ditampilkan, cuma hasil print-nya. | Tidak fatal, cuma output jadi kurang informatif dan double-print command mentah kalau `@` dihapus. |
| `bash infra/scripts/setup-kubeconfig.sh` | Eksekusi script 1: fetch & set kubeconfig. | `kubectl` di baris berikutnya tidak punya koneksi ke cluster → script kedua gagal connect. |
| `@echo "Applying node labels..."` | Print status transisi ke tahap 2. | Sama, hanya kehilangan info progress. |
| `bash infra/scripts/setup-node-label-taints.sh` | Eksekusi script 2: apply label & taint dari `config.yml`. | Kubeconfig ke-fetch tapi label/taint tidak pernah dipasang — tujuan utama command tidak tercapai. |

**Urutan wajib:** kubeconfig dulu, baru label/taint — karena script kedua manggil `kubectl label`/`kubectl taint`, yang butuh koneksi API server yang cuma tersedia setelah `KUBECONFIG` di-set.

**Kenapa tidak perlu `set -e` tambahan di Makefile:** Make otomatis stop kalau ada command di dalam target yang exit dengan status error. Kedua script juga sudah punya `set -e` sendiri di baris atasnya.

**Entri help:**
```makefile
@echo "  make setup-node-labels-taints - Fetch kubeconfig and apply node labels/taints from config.yml (standalone, for existing clusters)"
```
Murni dokumentasi, muncul saat `make help` dijalankan.

---

## 5. Cara Membuktikan Taint Berfungsi

Cek langsung:
```bash
kubectl describe node worker-2 | grep -A2 Taints
kubectl get node worker-2 -o jsonpath='{.spec.taints}'
```

Untuk membuktikan efeknya ke scheduler, deploy test pod **tanpa** toleration dengan `nodeSelector` ke `worker-2`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-taint
spec:
  nodeSelector:
    workload: db
  containers:
    - name: nginx
      image: nginx
```
```bash
kubectl apply -f test-taint-pod.yaml
kubectl get pod test-taint -o wide
kubectl describe pod test-taint | grep -A5 Events
```
Pod akan **Pending**, Events akan menunjukkan `untolerated taint`. Ini bukti taint mem-block scheduling walaupun `nodeSelector` sudah cocok.

Tambahkan `toleration` yang sesuai, apply ulang → Pod akan `Running`. Cleanup:
```bash
kubectl delete pod test-taint
```

**Konsep:** `nodeSelector`/label = menarik Pod ke node tertentu (opt-in dari sisi Pod). Taint = node menolak Pod yang tidak punya izin (opt-out), kecuali Pod punya `toleration` yang cocok.

---

## 6. Apakah `config.yml` Perlu Diupdate?

Untuk task ini: **tidak**. Data label/taint `worker-1` dan `worker-2` sudah ada sebelumnya dan dipakai apa adanya. Yang berubah cuma cara eksekusinya (jalur pintas), bukan datanya.

Tapi ini bukan aturan permanen. Kalau nanti ada node baru atau kebutuhan label/taint baru, `config.yml` tetap harus diedit manual dulu sebagai source of truth, baru `make setup-node-labels-taints` dijalankan untuk apply perubahan ke cluster.

---

## 7. Command Cek yang Perlu Disiapkan

**Docker:**
```bash
docker --version
docker ps
docker ps -a
sudo systemctl status docker
```

**Cluster / kubectl:**
```bash
echo $KUBECONFIG
kubectl config view --minify
kubectl get nodes -o wide
kubectl get nodes --show-labels
kubectl describe node worker-2 | grep -A2 Taints
kubectl get pods -A
kubectl get events -A --sort-by='.lastTimestamp'
```

**yq / config.yml:**
```bash
yq --version
yq '.infra.rke2.nodes' config.yml
```

**Makefile:**
```bash
make help | grep setup-node-labels-taints
grep -A6 "^setup-node-labels-taints:" Makefile
```

**Jalankan demo:**
```bash
make setup-node-labels-taints
```

---

## 8. Pertanyaan Jebakan Mentor + Jawaban

**Q: Kalau kubeconfig sebelumnya sudah pernah di-set, dijalankan lagi ada masalah?**
Tidak. `setup-kubeconfig.sh` selalu overwrite `~/.kube/config`, dan hapus dulu baris `export KUBECONFIG=` lama di `.bashrc` (`sed -i '/export KUBECONFIG=/d'`) sebelum menulis yang baru.

**Q: worker-1 kenapa tidak ada taint sama sekali?**
Disengaja — worker-1 diperuntukkan untuk workload aplikasi umum, jadi tidak perlu restriksi. Kalau dikasih taint juga, Pod aplikasi biasa (tanpa toleration) jadi tidak bisa dijadwalkan kemana-mana.

**Q: Kalau ada nama node duplikat di config.yml, script gimana?**
Loop berjalan berdasarkan index array, jadi `kubectl label`/`kubectl taint` akan dipanggil 2x ke node yang sama. Tidak error (karena ada `--overwrite`), cuma boros 1 iterasi.

**Q: Taint disimpan di resource apa?**
Field `.spec.taints` pada objek **Node**, bukan resource terpisah.
```bash
kubectl get node worker-2 -o jsonpath='{.spec.taints}'
```

**Q: Beda `NoSchedule` vs `NoExecute`?**
`NoSchedule`: cuma mencegah Pod baru dijadwalkan; Pod yang sudah jalan di situ tetap dibiarkan. `NoExecute`: Pod yang sudah jalan dan tidak punya toleration akan di-evict. Task ini pakai `NoSchedule` karena tujuannya cuma mengatur penjadwalan baru.

**Q: Dua script itu jalan di subshell terpisah per baris Makefile — kenapa `KUBECONFIG` dari script pertama bisa kebaca di script kedua?**
Karena `setup-kubeconfig.sh` menulis `KUBECONFIG` secara permanen ke `~/.kube/config` dan `~/.bashrc`, bukan cuma export sementara di shell-nya sendiri. Script kedua (`setup-node-label-taints.sh`) punya baris eksplisit `source ~/.bashrc` di awal, yang membaca ulang file itu meski dijalankan di subshell baru.
```bash
head -5 infra/scripts/setup-node-label-taints.sh
```

---

## 9. Kalimat Pembuka Demo

> "Bg, aku mau demo command baru buat pasang label dan taint ke cluster yang udah jalan duluan. Sebelumnya kalau client punya cluster sendiri dan belum di-label, kita harus manual set kubeconfig dulu baru bisa jalanin script label/taint-nya. Sekarang aku tambahin `make setup-node-labels-taints` yang otomatis fetch kubeconfig dulu, baru langsung apply label & taint dari config.yml. Aku mau tunjukin dari kondisi belum ada label sampai hasil akhir, sekalian buktiin taint-nya beneran ngefek ke scheduler pakai test pod."
