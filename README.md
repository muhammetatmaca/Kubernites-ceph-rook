# Kubernetes Kümesi Üzerinde Rook-Ceph Dağıtık Depolama Mimarisi

<div align="center">

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.28+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io)
[![Rook](https://img.shields.io/badge/Rook-Ceph%20Operator-red)](https://rook.io)
[![Ceph](https://img.shields.io/badge/Storage-Ceph%20Distributed-critical)](https://ceph.io)
[![Containerd](https://img.shields.io/badge/CRI-Containerd-575757)](https://containerd.io)
[![Author](https://img.shields.io/badge/Author-Muhammet%20Atmaca-black)](https://muhammetatmaca.com.tr)

**Bare-Metal & Sanallaştırılmış Düğümler Üzerinde Vanilla Kubernetes Kümesi, Flannel CNI, Rook-Ceph Dağıtık Blok/Dosya Depolama ve S3 Uyumlu MinIO Obje Depolama Entegrasyonu.**

[Teknik Dokümantasyon (PDF)](MuhammetAtmaca-Kubernetes.pdf) • [Geliştirici Portföyü](https://muhammetatmaca.com.tr) • [Mimari Şema](#-sistem-mimarisi-ve-kume-topolojisi) • [Test & Dogrulama](#-dogrulama-ve-failover-testleri)

</div>

---

## 1. Genel Bakış

Bu çalışma; yüksek erişilebilirlik (High Availability), hata toleransı ve dinamik veri kalıcılığı (Dynamic Volume Provisioning) sağlamak amacıyla oluşturulan kurumsal bir **Kubernetes + Rook-Ceph Dağıtık Depolama** altyapı projesidir.

Kubernetes üzerindeki durum bilgisi tutan (stateful) mikroservislerin ve bulut-yerel uygulamaların veri sürekliliğini garanti altına almak için, ham blok diskler Rook operatörü aracılığıyla Ceph depolama havuzuna (Storage Pool) dahil edilmiş; CSI (Container Storage Interface) sürücüleri üzerinden uygulamalara otomatik kalıcı birim (PersistentVolumeClaim - PVC) tahsis edilmiştir.

---

## 2. Sistem Mimarisi ve Küme Topolojisi

```
                               ┌───────────────────────────┐
                               │   Kubernetes Control Plane│
                               │   k8s-master (192.168.175.129)
                               │   API Server, etcd, Flannel │
                               └─────────────┬─────────────┘
                                             │
                      ┌──────────────────────┴──────────────────────┐
                      ▼                                             ▼
        ┌───────────────────────────┐                 ┌───────────────────────────┐
        │   k8s-node-1 (Worker 1)   │                 │   k8s-node-2 (Worker 2)   │
        │   IP: 192.168.175.131     │                 │   IP: 192.168.175.130     │
        │   RAM: 4 GB | OS: 40 GB   │                 │   RAM: 4 GB | OS: 40 GB   │
        │   Ceph Disk: /dev/sdb 100GB│                 │   Ceph Disk: /dev/sdb 100GB│
        └─────────────┬─────────────┘                 └─────────────┬─────────────┘
                      │                                             │
                      └──────────────────────┬──────────────────────┘
                                             │
                                             ▼
                          ┌─────────────────────────────────────┐
                          │    Rook-Ceph Storage Cluster Mesh   │
                          │   • Ceph MON, MGR & OSD Daemonlar   │
                          │   • 2x Replicated Pool (200 GB Raw) │
                          │   • Ceph Block Pool & CephFS        │
                          └──────────────────┬──────────────────┘
                                             │
                      ┌──────────────────────┴──────────────────────┐
                      ▼                                             ▼
        ┌───────────────────────────┐                 ┌───────────────────────────┐
        │ Stateful API Mikroservisi │                 │ S3 Uyumlu MinIO Gateway   │
        │ Flask App + Persistent PVC│                 │ Obje Depolama & Web Panel │
        └───────────────────────────┘                 └───────────────────────────┘
```

### Düğüm Donanım ve Ağ Konfigürasyonu

| Düğüm Adı | Rolü | Statik IP | vCPU / RAM | Sistem Diski | Ceph OSD Veri Diski |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **k8s-master** | Control Plane | `192.168.175.129` | 2 Core / 2 GB | 20 GB | - |
| **k8s-node-1** | Worker Node | `192.168.175.131` | 2 Core / 4 GB | 40 GB | **/dev/sdb (100 GB Raw SSD)** |
| **k8s-node-2** | Worker Node | `192.168.175.130` | 2 Core / 4 GB | 40 GB | **/dev/sdb (100 GB Raw SSD)** |

---

## 3. Katmanlı Uygulama Mimarisi

### 3.1. Altyapı ve Kubernetes Kümeleme (Bootstrap)
1. **Container Runtime:** Docker yerine güncel standart olan `containerd` çalışma zamanı kurulmuş ve `SystemdCgroup = true` yapılandırması uygulanmıştır.
2. **Küme Başlatma:** `kubeadm init --pod-network-cidr=10.244.0.0/16` ile master düğüm oluşturulmuş; worker düğümleri şifreli token ile kümeye eklenmiştir.
3. **CNI Ağ Katmanı:** Düğümler arası pod haberleşmesi için `Flannel CNI` uygulanmıştır.

### 3.2. Rook Operatörü ile Ceph Orkestrasyonu
* **Cloud-Native Depolama Operatörü:** Kubernetes CRD (Custom Resource Definition) mekanizması kullanan Rook operatörü dağıtılmıştır.
* **Otomatik OSD Keşfi:** Worker düğümlerine takılan biçimlendirilmemiş 100 GB'lık `/dev/sdb` ikincil diskleri taranarak `Ceph OSD (Object Storage Daemon)` podlarına dönüştürülmüştür.
* **StorageClass & Dynamic Provisioning:** Uygulamaların talebine göre anında hacim üreten `rook-ceph-block` depolama sınıfı tanımlanmıştır.

### 3.3. Kalıcı Durumlu Mikroservis ve S3 MinIO Katmanı
* **Stateful File API:** Base64 formatında dosya yükleme/indirme yapan Python/Flask tabanlı REST servisi geliştirilmiş ve Dockerize edilmiştir.
* **PVC Entegrasyonu:** Uygulama podunun `/data` dizini Ceph CSI üzerinden sağlanan PVC'ye bağlanmıştır.
* **MinIO Obje Depolama:** S3 uyumlu API uç noktası sağlamak amacıyla Ceph kalıcı hacmi üzerinde MinIO sunucusu çalıştırılmıştır.

---

## 4. Doğrulama ve Failover Testleri

Depolama kümesinin dayanıklılığı ve hata toleransı şu senaryolarla test edilmiş ve doğrulanmıştır:

1. **Pod Yeniden Başlatma & Yeniden Dağıtım (Resilience Test):**
   * Mikroservis üzerinden Ceph kalıcı birimine dosyalar yüklendi.
   * `kubectl delete pod` komutuyla pod zorla sonlandırıldı.
   * Yeni açılan pod aynı PVC'ye bağlandı ve tüm verilerin eksiksiz olarak korunduğu doğrulandı.

2. **Düğüm Arızası ve Veri Kurtarma (Node Eviction Test):**
   * Worker 1 üzerinde çalışan pod kapatıldığında, Kubernetes pod'u Worker 2 düğümüne taşıdı.
   * Ceph'in replike veri dağıtım mimarisi sayesinde veri kaybı ve I/O kilitlenmesi yaşanmadı.

3. **MinIO S3 Obje İletişimi:**
   * Web arayüzü ve AWS CLI/S3 API aracılığıyla bucket oluşturma ve obje yükleme testleri başarıyla tamamlandı.

---

## 5. Dağıtım ve Kurulum Adımları

### 1. Düğümleri Hazırlayın
Tüm sunucularda swap alanını kapatın ve kernel parametrelerini yükleyin:
```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

### 2. Rook-Ceph Operatörünü Dağıtın
```bash
git clone --single-branch --branch v1.12.0 https://github.com/rook/rook.git
cd rook/deploy/examples

kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml
```

### 3. Küme Durumunu İzleyin
```bash
kubectl -n rook-ceph get pod
kubectl -n rook-ceph get cephcluster
```

---

## 6. Proje Materyalleri

* **Detaylı Rapor ve Teknik Dokümantasyon:** Projenin adım adım konsol çıktıları, mimari diyagramları ve analizlerini içeren tam metin PDF dosyası repo içerisinde yer almaktadır: [`MuhammetAtmaca-Kubernetes.pdf`](MuhammetAtmaca-Kubernetes.pdf).

---

## 7. Geliştirici ve İletişim

**Muhammet Atmaca**  
* **Web Sitesi:** [muhammetatmaca.com.tr](https://muhammetatmaca.com.tr)  
* **GitHub:** [@muhammetatmaca](https://github.com/muhammetatmaca)  
* **E-posta:** info@muhammetatmaca.com.tr  

---

## 8. Lisans

Bu proje [MIT](LICENSE) lisansı altında sunulmaktadır.
