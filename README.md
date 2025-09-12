# Kubernetes & Ceph Entegrasyon Projesi
![Kubernetes Architecture](https://cdn-dynmedia-1.microsoft.com/is/image/microsoftcorp/what-is-kubernetes_kubernetes-community?resMode=sharp2&op_usm=1.5,0.65,15,0&wid=1062&hei=620&qlt=100&fit=constrain)

### 🚀 Proje Hakkında
Bu proje, yüksek erişilebilirlik ve ölçeklenebilirlik sağlamak amacıyla bir **Ceph dağıtık depolama kümesinin**, **Kubernetes** ortamı üzerine kurulmasını ve entegrasyonunu ele alır. Geliştirilen basit bir API uygulaması, Ceph'in sağladığı kalıcı depolama alanını (Persistent Volume Claim - PVC) kullanarak verilerini kalıcı hale getirir.

Proje, dağıtık sistemler ve bulut teknolojileri alanındaki teorik bilgileri pratik bir senaryo ile pekiştirmeyi hedefler.

---

### 🌟 Proje Topolojisi
Proje, bir Master ve iki Worker düğümünden oluşan basit bir küme topolojisine sahiptir.

| Rol | Makine Adı | IP Adresi | RAM | Diskler |
| :--- | :--- | :--- | :--- | :--- |
| **Master** | k8s-master | 192.168.175.129 | 2 GB | 20 GB |
| **Node 1** | k8s-node-1 | 192.168.175.131 | 4 GB | 40 GB + **100 GB** |
| **Node 2** | k8s-node-2 | 192.168.175.130 | 4 GB | 40 GB + **100 GB** |

*Her bir Worker düğümündeki **100 GB'lık ikinci diskler**, Ceph depolama havuzu için kullanılmıştır.*

---

### 📦 Bölüm 1: Altyapı ve Kubernetes Kurulumu
Bu bölümde, proje için gerekli sanal makineler kurulmuş ve Kubernetes kümesi yapılandırılmıştır.

1.  **Sanal Makine Kurulumu:** VirtualBox üzerinde master ve iki worker makinesi oluşturuldu. Worker makinelerine Ceph için ek diskler eklendi.
2.  **Statik IP Ataması:** Tüm makinelere sabit IP adresleri atanarak ağ iletişimi sağlandı.
3.  **Ortak Paket Kurulumları:** Tüm makinelere `containerd`, `kubeadm`, `kubelet` ve `kubectl` gibi temel paketler kuruldu.
4.  **Kubernetes Küme Başlatma:** `kubeadm init` komutu ile master düğümde küme başlatıldı ve `kubeadm join` ile worker düğümleri kümeye dahil edildi. `Flannel` CNI (Container Network Interface) eklentisi kuruldu.

---

### 📝 Bölüm 2: API Uygulaması ve Dağıtım
Bu aşamada, dosya yükleme ve indirme işlemleri için basit bir Python API geliştirildi ve Kubernetes'te çalıştırılmaya hazır hale getirildi.

1.  **API Geliştirme:** Base64 formatında veri alıp kaydeden bir **Flask** API (`app.py`) oluşturuldu.
2.  **Dockerizasyon:** Uygulama, `Dockerfile` ile bir Docker imajına dönüştürüldü (`my-simple-api:v1`).
3.  **Kubernetes Dağıtımı:** Uygulama, bir **Deployment** ve **Service** objeleri kullanılarak Kubernetes üzerinde çalıştırıldı.

---

### 🛡️ Bölüm 3: Ceph ve Kalıcı Depolama
Bu kısım, projenin kalbi olan Ceph entegrasyonunu ve Minio kurulumunu içerir.

1.  **Rook Operatörü ile Ceph Kurulumu:** **Rook** operatörü, Ceph'in Kubernetes'e kolayca dağıtılması ve yönetilmesi için kullanıldı. `cluster.yaml` dosyası ile nodelardaki diskler otomatik olarak Ceph depolamasına dahil edildi.
2.  **Kalıcı Depolama (PVC):** API uygulaması, Ceph'in sağladığı depolama alanını bir `PersistentVolumeClaim` (PVC) ile talep etti. Bu sayede, pod'lar yeniden başlatılsa bile veriler kalıcı oldu.
3.  **Minio Kurulumu:** Ceph'in üzerine, S3 uyumlu bir obje depolama çözümü olan **Minio** kuruldu. Minio da kendi verileri için Ceph'in kalıcı depolama alanını kullandı.

---

### ✅ Testler ve Sonuçlar
* API'ye `cURL` komutları ile dosya yükleme ve indirme işlemleri yapıldı.
* API'nin pod'u silinip yeniden oluşturularak kalıcı depolama testi başarıyla gerçekleştirildi. Dosyaların kaybolmadığı teyit edildi.
* Minio web arayüzüne erişilerek obje depolama işlevselliği test edildi.
