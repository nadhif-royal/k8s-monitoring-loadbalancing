# High Availability Kubernetes Cluster (Capstone Project)

## Identitas Kelompok
* **Muhammad Habibi** (235150201111063)


1. Implementasi Monitoring & Visualisasi: Melakukan instalasi serta konfigurasi Metrics Server dan Grafana untuk memantau performa resource (CPU/RAM) seluruh node dan pod secara real-time.

2. Optimasi High Availability: Melakukan troubleshooting pada Metrics Server dan mengonfigurasi Horizontal Pod Autoscaler (HPA) dengan target utilisasi CPU 80% untuk memastikan skalabilitas otomatis aplikasi.

3. Verifikasi Load Balancing: Mengelola distribusi trafik melalui Service Kubernetes dan melakukan pengujian mekanisme Round Robin untuk memastikan beban kerja terbagi rata ke seluruh replika Pod (verifikasi via header X-Served-By).

4. Infrastruktur & Storage: Bertanggung jawab atas inisialisasi cluster (1 Master & 3 Workers), konfigurasi jaringan Calico CNI, serta manajemen Persistent Storage MySQL menggunakan PV/PVC.
* **Nadhif Rif’at Rasendriya** (235150201111074)

1. Konseptor Topologi & Arsitektur: Merancang struktur topologi jaringan lokal, alokasi IP statis untuk Control Plane dan Data Plane, serta menyusun strategi deployment.

2. Technical Writer & Documentation: Menyusun blueprint panduan instalasi (README.md) dan merumuskan dokumentasi analitis terhadap hasil implementasi sistem.

3. Quality Assurance & Troubleshooting: Membantu analisis pemecahan masalah (troubleshooting) secara konseptual selama proses integrasi node dan memvalidasi konfigurasi beban kerja (load balancing) pada Pods.

---

## Arsitektur Jaringan & Node
Cluster ini berjalan menggunakan jaringan lokal (FILKOM) dengan konfigurasi IP statis menggunakan **Bridge Adapter** pada VirtualBox:

| Node Name | Role | IP Address | Operating System |
| :--- | :--- | :--- | :--- |
| **Master-Node** | Control Plane | `192.168.1.13` | Ubuntu 22.04/24.04 |
| **Worker-1** | Data Plane | `192.168.1.10` | Ubuntu 22.04/24.04 |
| **Worker-2** | Data Plane | `192.168.1.11` | Ubuntu 22.04/24.04 |
| **Worker-3** | Data Plane | `192.168.1.12` | Ubuntu 22.04/24.04 |

---

## Panduan Instalasi (Technical Setup)

### 1. Persiapan Docker Engine & Repository
```bash
wget -O - [https://download.docker.com/linux/ubuntu/gpg](https://download.docker.com/linux/ubuntu/gpg) > ./docker.key
gpg --no-default-keyring --keyring ./docker.gpg --import ./docker.key
gpg --no-default-keyring --keyring ./docker.gpg --export > ./docker-archive-keyring.gpg
sudo mv ./docker-archive-keyring.gpg /etc/apt/trusted.gpg.d/
sudo add-apt-repository "deb [arch=amd64] [https://download.docker.com/linux/ubuntu](https://download.docker.com/linux/ubuntu) $(lsb_release -cs) stable" -y
sudo apt update -y
sudo apt install git wget curl socat docker-ce -y
```

### 2. Instalasi cri-dockerd (Docker CRI Support)
```bash
VER=$(curl -s [https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest](https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest)|grep tag_name | cut -d '"' -f 4|sed 's/v//g')
wget [https://github.com/Mirantis/cri-dockerd/releases/download/v$](https://github.com/Mirantis/cri-dockerd/releases/download/v$){VER}/cri-dockerd-${VER}.amd64.tgz
tar xzvf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
wget [https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.service)
wget [https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket](https://raw.githubusercontent.com/Mirantis/cri-dockerd/master/packaging/systemd/cri-docker.socket)
sudo mv cri-docker.socket cri-docker.service /etc/systemd/system/
sudo sed -i -e 's,/usr/bin/cri-dockerd,/usr/local/bin/cri-dockerd,' /etc/systemd/system/cri-docker.service
sudo systemctl daemon-reload
sudo systemctl enable cri-docker.service
sudo systemctl enable --now cri-docker.socket
```

### 3. Instalasi Kubernetes & Optimasi Sistem
```bash
curl -fsSL [https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key](https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key) | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] [https://pkgs.k8s.io/core:/stable:/v1.31/deb/](https://pkgs.k8s.io/core:/stable:/v1.31/deb/) /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold docker-ce kubelet kubeadm kubectl
sudo swapoff -a
```

---

## Deployment Aplikasi (k8s-login-app)
Aplikasi dideploy dengan **3 Replicas** yang disebar secara otomatis oleh *Kube-Scheduler* ke ketiga Worker Node untuk menjamin *High Availability*.

```bash
cd k8s-login-app/k8s
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-pv.yaml
kubectl apply -f mysql-pvc.yaml
kubectl apply -f mysql-service.yaml
kubectl apply -f mysql-deployment.yaml
kubectl apply -f web-service.yaml
kubectl apply -f web-deployment.yaml
```

---

## Bukti Implementasi dan Analisis

### Gambar 1. Status Operasional Monitoring Stack
**Penjelasan:** Melalui eksekusi perintah `kubectl get pods -n monitoring -o wide`, dilakukan verifikasi terhadap kesiapan seluruh komponen *monitoring stack*. Hasil menunjukkan bahwa seluruh *pod* esensial meliputi Prometheus, Grafana, Alertmanager, dan Node Exporter, telah berada dalam status `Running` dan tersebar secara merata di seluruh *node* (Master dan ketiga Worker). Distribusi ini membuktikan keberhasilan konfigurasi *high availability* pada lapisan pemantauan klaster.

### Gambar 2. Verifikasi Akses Antarmuka Grafana
**Penjelasan:** Gambar ini menampilkan halaman autentikasi Grafana yang berhasil diakses melalui peramban web eksternal. Keberhasilan akses ini mengonfirmasi bahwa *service* Grafana telah terekspos dengan benar menggunakan mekanisme *NodePort* atau *LoadBalancer*. Hal ini memastikan bahwa operator sistem dapat melakukan pengawasan visual terhadap metrik klaster secara *real-time* dari luar jaringan klaster.

### Gambar 3. Analisis Telemetri Resource Control Plane
**Penjelasan:** Visualisasi dasbor Grafana ini menyajikan metrik penggunaan CPU dan memori secara spesifik pada mesin *Control Plane*. Data telemetri yang dikumpulkan oleh `node-exporter` dan disimpan dalam pangkalan data deret waktu (*time-series*) Prometheus berhasil ditampilkan secara akurat. Pemantauan pada *node* ini kritikal untuk memastikan stabilitas komponen inti klaster seperti API Server, *Scheduler*, dan *Controller Manager*.

### Gambar 4. Monitoring Resource Terdistribusi pada Worker-1
**Penjelasan:** Dokumentasi ini menyajikan pemantauan penggunaan sumber daya pada *Worker-1*. Analisis terhadap grafik ini menunjukkan konsumsi memori dan CPU yang stabil. Kemampuan untuk memilah data metrik per *node* membuktikan efektivitas sistem *monitoring* dalam mendeteksi anomali atau ketimpangan beban kerja (*resource imbalance*) di seluruh infrastruktur klaster.

### Gambar 5. Monitoring Resource Terdistribusi pada Worker-2
**Penjelasan:** Serupa dengan node lainnya, pemantauan pada *Worker-2* menunjukkan performa yang konsisten. Visualisasi ini memastikan bahwa beban kerja yang dijalankan pada node ini terpantau dengan baik oleh sistem Prometheus.

### Gambar 6. Monitoring Resource Terdistribusi pada Worker-3
**Penjelasan:** Pemantauan pada *Worker-3* melengkapi visibilitas infrastruktur secara menyeluruh. Konsistensi data di seluruh node pelaksana tugas membuktikan keandalan *node-exporter* dalam mengumpulkan data telemetri.

### Gambar 7. Verifikasi High Availability dan Distribusi Pod Aplikasi
**Penjelasan:** Berdasarkan *output* `kubectl get pods -o wide`, terlihat bahwa tiga replika dari aplikasi `login-app` telah dideploy dan aktif. Penempatan *pod* yang tersebar secara otomatis oleh *Kube-scheduler* ke Worker-1, Master, dan Worker-2 merupakan implementasi nyata dari prinsip *fault tolerance*. Jika salah satu *node* mengalami kegagalan, layanan tetap tersedia melalui replika di *node* lainnya.

### Gambar 8. Monitoring Health Check via Prometheus Service Discovery
**Penjelasan:** Tampilan antarmuka 'Targets' pada Prometheus memvalidasi fungsionalitas `ServiceMonitor` yang telah dikonfigurasi. Sistem berhasil melakukan penemuan layanan (*service discovery*) terhadap tiga titik akhir (*endpoints*) dari `login-app`. Status `UP` pada seluruh target menandakan bahwa Prometheus sukses melakukan penarikan metrik (*scraping*) dari aplikasi untuk diolah lebih lanjut.

### Gambar 9. Observabilitas Kinerja Aplikasi Tingkat Pod
**Penjelasan:** Dasbor kustom ini memberikan visibilitas mendalam terhadap performa spesifik pod `login-app`, meliputi konsumsi CPU, penggunaan memori, serta lalu lintas jaringan masuk dan keluar. Melalui visualisasi ini, pengembang dapat menganalisis efisiensi kode dan dampak operasional aplikasi terhadap infrastruktur secara granulasi, yang sangat penting untuk proses optimasi performa.

### Gambar 10. Analisis Baseline Performance dan Inisialisasi HPA
**Penjelasan:** Pada tahap inisialisasi, dilakukan pengecekan beban awal menggunakan `kubectl top nodes` dan pemantauan status Horizontal Pod Autoscaler (HPA). Kondisi awal menunjukkan beban CPU yang rendah (27%) dengan target ambang batas autoscaling yang ditetapkan pada 80%. Tahap ini sangat penting sebagai titik acuan (*baseline*) untuk menguji reaktivitas sistem otomatisasi terhadap lonjakan beban nantinya.

### Gambar 11. Skenario Injeksi Beban Kerja (Traffic Generation)
**Penjelasan:** Dokumentasi ini memperlihatkan pengoperasian *pod* `load-gen` yang dirancang untuk mensimulasikan lonjakan trafik pengguna secara intensif ke aplikasi `login-app`. Pengecekan status *pod* secara berdampingan memastikan bahwa simulasi beban sedang berlangsung secara aktif sementara aplikasi sasaran terus memberikan layanan.

### Gambar 12. Analisis Reaktivitas HPA terhadap Lonjakan Beban CPU
**Penjelasan:** Gambar ini menangkap momen kritis di mana intensitas trafik dari `load-gen` berhasil memicu lonjakan penggunaan CPU hingga 87%, melampaui batas aman 80% yang telah dikonfigurasi. Kondisi ini membuktikan bahwa HPA telah berhasil mendeteksi beban berlebih secara dinamis dan bersiap untuk menjalankan instruksi *scale-out* (penambahan kapasitas) guna menjaga stabilitas layanan.

### Gambar 13. Analisis Stabilitas Klaster dan Fase Cool-down HPA
**Penjelasan:** Pasca penghentian simulasi beban, metrik penggunaan CPU terpantau menurun secara signifikan ke angka 18%. Hasil ini mendemonstrasikan kemampuan klaster dalam melakukan normalisasi mandiri. Sistem HPA menunjukkan stabilitas operasional dengan memantau penurunan beban sebelum nantinya melakukan *scale-in* (pengurangan replika) secara otomatis untuk mengefisiensikan penggunaan sumber daya.

---
**Link Github:** [https://github.com/nadhif-royal/k8s-monitoring-loadbalancing](https://github.com/nadhif-royal/k8s-monitoring-loadbalancing)
