---
layout: default
title: Kubernetes Troubleshooting
nav_order: 3
description: "Kubernetes cluster yönetimi ve sorun giderme için pratik komutlar ve çözümler"
permalink: /kubernetes-troubleshooting
---

# Kubernetes Troubleshooting Rehberi
{: .no_toc }

Kubernetes cluster yönetimi ve sorun giderme için kapsamlı komut ve çözüm rehberi.
{: .fs-6 .fw-300 }

## İçindekiler
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 🔧 Node İşlemleri

### 🔹 Node'u Klasik Yöntemle Küme'ye Ekleme

```bash
kubeadm join <MASTER_IP>:<PORT> --token <TOKEN> --discovery-token-ca-cert-hash sha256:<HASH>
```

- `kubeadm token create --print-join-command` komutuyla master'dan join komutunu alabilirsin.

### 🔹 Node'un Sağlığını Kontrol Etme

```bash
kubectl get nodes
kubectl describe node <NODE_ADI>
```

- `NotReady` durumu görüyorsan genelde `kubelet` ya da `container runtime` çalışmıyordur.
- SSH ile node’a girip `systemctl status kubelet` ile bak.

### 🔹 Node'dan Taint Kaldırma

```bash
kubectl taint nodes <NODE_ADI> node-role.kubernetes.io/master-
```

- Eğer `NoSchedule` nedeniyle Pod’lar bu node’a gitmiyorsa bu komutla açarsın.

### 🔹 Node’a Taint Ekleme

```bash
kubectl taint nodes <NODE_ADI> key=value:NoSchedule
```

- Pod’ların bu node’a gitmesini istemiyorsan (örneğin özel işler için) kullanılır.

## 🔄 Pod Yeniden Başlatma

```bash
kubectl delete pod <POD_ADI> -n <NAMESPACE>
```

- Deployment varsa otomatik yeniden gelir.
- ConfigMap veya Secret değiştiğinde restart gerekebilir.

## 🧹 Pod Temizleme

```bash
kubectl get pods --all-namespaces | grep Evicted
kubectl delete pod <POD_ADI> -n <NAMESPACE>
```

- Evicted pod’lar sistemi kirletir, düzenli temizle.

## 🔍 Pod Loglarını İnceleme

```bash
kubectl logs <POD_ADI> -n <NAMESPACE>
kubectl logs <POD_ADI> -c <CONTAINER_ADI> -n <NAMESPACE>
```

- Hata trace’lerini görmek için birebir.
- Çok eski loglar için `kubectl logs --previous` kullanılabilir.

## 🔁 Pod içinde Komut Çalıştırma

```bash
kubectl exec -it <POD_ADI> -n <NAMESPACE> -- /bin/sh
```

- Bazı imajlarda `/bin/bash` yerine `/bin/sh` çalışır.

## 🌐 Servis (Service) Problemleri

### 🔹 Service Ulaşılabilirlik Testi

```bash
kubectl get svc -n <NAMESPACE>
kubectl describe svc <SERVICE_ADI> -n <NAMESPACE>
```

- Servis `ClusterIP`, `NodePort`, `LoadBalancer` tiplerinden hangisi?
- Hedef Pod’lar gerçekten ayakta mı?

### 🔹 DNS Çözümleme Problemi

```bash
kubectl exec -it <POD_ADI> -- nslookup <SERVICE_ADI>.<NAMESPACE>.svc.cluster.local
```

- CoreDNS problemi olabilir. Aşağıdaki gibi yeniden başlatılabilir:

```bash
kubectl -n kube-system rollout restart deployment coredns
```

## 📦 PVC (PersistentVolumeClaim) Problemleri

### 🔹 PVC Durumlarını Görüntüleme

```bash
kubectl get pvc -n <NAMESPACE>
kubectl describe pvc <PVC_ADI> -n <NAMESPACE>
```

- `Pending` durumda kalıyorsa uygun StorageClass yok demektir.
- `kubectl get sc` ile mevcut sınıfları kontrol et.

## 💥 CrashLoopBackOff Sorunları

### 🔹 Pod Detaylarına Bak

```bash
kubectl describe pod <POD_ADI> -n <NAMESPACE>
```

- Olayları (Events) incele. Genellikle yanlış environment variable, configmap hatası çıkar.

### 🔹 Container Logları

```bash
kubectl logs <POD_ADI> --previous -n <NAMESPACE>
```

- `--previous`, çöken son container’ın logunu gösterir.

## 🚨 Pod Scheduling Problemleri

```bash
kubectl describe pod <POD_ADI>
```

- `Events` kısmında `0/3 nodes are available: 3 Insufficient cpu` gibi mesajlar varsa kaynak yetersiz.
- Yeni node eklemek ya da kaynak limitlerini düşürmek gerekebilir.

## 📡 LoadBalancer Servis Dış IP Alamıyor

```bash
kubectl get svc <SERVICE_ADI>
```

- Dış IP gelmiyorsa `cloud controller manager` loglarına bak.
- `MetalLB` gibi bare-metal çözümlerde config hatası olabilir:

```bash
kubectl get configmap -n metallb-system config -o yaml
```

## 🚀 Deployment Sorunları

### 🔹 Rollout Durumunu Kontrol Et

```bash
kubectl rollout status deployment <DEPLOYMENT_ADI> -n <NAMESPACE>
```

- Deployment takıldıysa, yeni imaj çekilememiş olabilir. ImagePullBackOff hatasına dikkat.

### 🔹 Deployment’ı Geri Alma (Rollback)

```bash
kubectl rollout undo deployment <DEPLOYMENT_ADI> -n <NAMESPACE>
```

- Yeni sürüm sorun çıkardıysa bir önceki haline döner.

## 🛠️ ConfigMap Sorunları

### 🔹 ConfigMap'i Yeniden Yüklemek

```bash
kubectl create configmap <AD> --from-file=dosya.env -o yaml --dry-run=client | kubectl apply -f -
```

- ConfigMap değişti ama Pod hala eski halindeyse Pod’u yeniden başlatmak gerekebilir.

### 🔹 Pod'da ConfigMap’in Yüklendiğini Kontrol Et

```bash
kubectl exec -it <POD_ADI> -n <NAMESPACE> -- cat /path/in/container/config.env
```

## 🔐 Secret Problemleri

### 🔹 Secret’i Okumak (Base64 Decode ile)

```bash
kubectl get secret <SECRET_ADI> -n <NAMESPACE> -o jsonpath="{.data}"
```

- Çıkan değerler base64 ile encode’dur.

```bash
echo "cGFzc3dvcmQ=" | base64 -d
```

### 🔹 Secret’i Güncellemek

```bash
kubectl create secret generic <AD> --from-literal=key=value -o yaml --dry-run=client | kubectl apply -f -
```

## 🧠 Faydalı `kubectl` Kısayolları

### 🔹 Etiketle Node veya Pod

```bash
kubectl label nodes <NODE_ADI> zone=istanbul
kubectl label pods <POD_ADI> env=prod
```

### 🔹 Kaynakları Etiket ile Filtrele

```bash
kubectl get pods -l env=prod
```

### 🔹 Sık Kullanılan `kubectl` Kısa Kodu

```bash
kubectl get all -n <NAMESPACE>
```

- Tüm kaynakları (pod, svc, deploy vs) bir anda listeler.

## 🔐 NetworkPolicy Sorunları

### 🔹 Pod'lar Arası Erişim Engelleniyor

```bash
kubectl get networkpolicy -n <NAMESPACE>
kubectl describe networkpolicy <POLICY_ADI> -n <NAMESPACE>
```

- Erişim engellenmişse `allow all` politikası test için yazılabilir:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all
  namespace: default
spec:
  podSelector: {}
  ingress:
    - {}
  egress:
    - {}
  policyTypes:
    - Ingress
    - Egress
```

## ⚙️ Helm Sorunları

### 🔹 Helm Release’leri Görüntüleme

```bash
helm list -A
```

- Tüm namespace'lerdeki release’leri gösterir.

### 🔹 Helm ile Kurulu Kaynağı Silmek

```bash
helm uninstall <RELEASE_ADI> -n <NAMESPACE>
```

- Helm ile kuruluysa `kubectl delete` ile silmek eksik temizlik yapabilir.

## 🌍 Ingress Problemleri

### 🔹 Ingress Kaynaklarını Görmek

```bash
kubectl get ingress -A
```

- IP/hostname çözülmüyorsa DNS kaydı eksik olabilir.
- Ingress controller (örneğin nginx) çalışıyor mu?

```bash
kubectl get pods -n ingress-nginx
```

## 📊 Metrics Server Sorunları

### 🔹 `kubectl top` Çalışmıyor

```bash
kubectl top nodes
```

- Hata alıyorsan Metrics Server kurulu değil demektir.
- Kurulum sonrası `--kubelet-insecure-tls` parametresi gerekebilir:

```bash
--kubelet-insecure-tls
```

### 🔹 Metrics Server Pod Loglarına Bak

```bash
kubectl logs <POD_ADI> -n kube-system
```

## 📁 Pod ile Dosya Transferi (kubectl cp)

### 🔹 Pod'dan Host'a Dosya Kopyalama

```bash
kubectl cp <NAMESPACE>/<POD_ADI>:/path/in/container /local/path
```

### 🔹 Host’tan Pod’a Dosya Kopyalama

```bash
kubectl cp /local/path <NAMESPACE>/<POD_ADI>:/path/in/container
```

- Pod çalışır durumda olmalı. Hedef yol mevcut değilse hata verir.

## 🧬 etcd Backup ve Geri Yükleme

### 🔹 etcd Yedeği Alma (Kubeadm Kurulumları İçin)

```bash
ETCDCTL_API=3 etcdctl snapshot save snapshot.db   --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key
```

### 🔹 etcd Yedeği Geri Yükleme

```bash
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db   --data-dir=/var/lib/etcd-from-backup
```

- Dikkat: Mevcut etcd klasörünü yedekleyip kapatmadan restore yapma.

## ⚙️ Kube-Proxy Problemleri

### 🔹 Kube-Proxy Loglarını İncele

```bash
kubectl -n kube-system logs -l k8s-app=kube-proxy
```

- Servis routing problemi varsa burada ipucu bulabilirsin.

### 🔹 iptables Kuralları Doğru mu?

```bash
iptables -t nat -L -n | grep KUBE
```

- Kube-proxy `iptables` modundayken bu kuralları yazar. Eksikse proxy çalışmıyor olabilir.

## 🧩 CNI (Container Network Interface) Sorunları

### 🔹 Pod’lar Arası Erişim Yoksa

- `kubectl get pods -o wide` ile IP’leri kontrol et.
- CNI plugin (Flannel, Calico, vs.) podları çalışıyor mu?

```bash
kubectl get pods -n kube-system
```

### 🔹 Yeni Node Ekledin, Ama Pod’lar IP Almıyor

- CNI plugin node üzerinde düzgün yüklenmemiş olabilir.
- Node’da aşağıdakileri kontrol et:

```bash
ls /etc/cni/net.d/
```

## 🌐 CoreDNS Sorunları

### 🔹 CoreDNS Pod’larını Görüntüle

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

- `CrashLoopBackOff` durumundaysa `ConfigMap` hatalı olabilir:

```bash
kubectl -n kube-system edit configmap coredns
```

## 🔒 API Server Ulaşılabilirlik

### 🔹 Apiserver Durumu

```bash
kubectl get componentstatuses
```

- `etcd`, `scheduler`, `controller-manager`, `apiserver` buradan izlenebilir.

### 🔹 API Server Logları

```bash
journalctl -u kube-apiserver
```

- Hata durumlarında burada detaylı log bulursun.

## 🧠 Kubelet Sorunları

### 🔹 Kubelet Hizmeti Çalışıyor mu?

```bash
systemctl status kubelet
```

- `Failed` ise genellikle yanlış config veya `/etc/kubernetes/kubelet.conf` dosyası bozulmuştur.

### 🔹 Kubelet Logları

```bash
journalctl -u kubelet
```

## 🔥 Düşük Seviyeli Network Problemleri

### 🔹 iptables ya da ipvs Doğrulama

```bash
iptables-save | grep KUBE
ipvsadm -Ln
```

- Pod'lara trafik yönlenmiyorsa burada sıkıntı olabilir.

## 📝 Günlük Kayıtları (systemd journal)

### 🔹 Belirli Süre İçin Kayıtları Görüntüle

```bash
journalctl -u kubelet --since "2 hours ago"
```

### 🔹 En Son Kayıtları Canlı Takip Etmek

```bash
journalctl -u kubelet -f
```

## 📥 Log Toplama & İnceleme

### 🔹 Namespace Bazlı Tüm Pod Logları

```bash
for pod in $(kubectl get pods -n <NAMESPACE> -o jsonpath="{.items[*].metadata.name}"); do
  echo "== $pod ==";
  kubectl logs $pod -n <NAMESPACE>;
done
```

- Geniş hata taraması için faydalı.

### 🔹 Crashloop Olaylarını Çekmek

```bash
kubectl get events --all-namespaces | grep -i crash
```

## ⛔ Node Drain & Cordoning

### 🔹 Node Drain (Pod'ları Taşıyarak Boşaltma)

```bash
kubectl drain <NODE_ADI> --ignore-daemonsets --delete-emptydir-data
```

- Genelde bakım veya silme öncesi yapılır.
- StatefulSet varsa dikkatli olunmalı, veri kaybı yaşanabilir.

### 🔹 Node'u Scheduling’e Kapatmak

```bash
kubectl cordon <NODE_ADI>
```

### 🔹 Node Scheduling’i Açmak

```bash
kubectl uncordon <NODE_ADI>
```

## 🧱 Takılı Kalmış PVC/PV Problemleri

### 🔹 PVC `Terminating` Durumunda Kalıyor

```bash
kubectl get pvc -n <NAMESPACE>
```

- Finalizer’ı temizlemek gerekebilir:

```bash
kubectl patch pvc <PVC_ADI> -n <NAMESPACE> -p '{"metadata":{"finalizers":null}}'
```

### 🔹 PV `Released` Durumunda Kalıyor

- Manüel olarak temizlenmeli ya da reclaim policy `Retain` ise silinip yeniden oluşturulmalı.

## 🧲 Pod Affinity / Anti-Affinity Sorunları

### 🔹 Scheduling Neden Olmuyor?

```bash
kubectl describe pod <POD_ADI>
```

- `MatchExpressions` veya `requiredDuringSchedulingIgnoredDuringExecution` kuralları hatalı olabilir.
- Etiket uyuşmazlığı varsa `0/3 nodes match pod affinity rules` şeklinde uyarı verir.

## ⬆️ Upgrade Sonrası Sorunlar

### 🔹 API Version Deprecated (Örneğin apps/v1beta1)

```bash
kubectl get deploy -o yaml | grep apiVersion
```

- API değişikliklerinde önce tüm CRD ve manifest'ler yeni versiyona uyumlu hale getirilmeli.

### 🔹 CoreDNS, kube-proxy veya CNI Uyuşmazlıkları

- `kubectl get ds -n kube-system` ile `DaemonSet` durumlarını kontrol et.
- Yeni versiyona uygun imaj tag’leri kullanıldığından emin olun.

## 🧩 Custom Resource Definition (CRD) Sorunları

### 🔹 CRD’yi Listele ve Detaylarını Gör

```bash
kubectl get crd
kubectl describe crd <CRD_ADI>
```

- CRD eksikse ona bağlı controller çalışmaz.
- CRD versiyonu (v1beta1 vs v1) uyumsuzluğu problem yaratabilir.

### 🔹 CRD ile Oluşturulan Kaynaklar

```bash
kubectl get <KAYNAK_TÜRÜ> -n <NAMESPACE>
```

- Örneğin: `kubectl get prometheuses.monitoring.coreos.com -n monitoring`

## 🔔 Mutating/Validating Webhook Sorunları

### 🔹 Webhook’lar Neden Sorun Çıkarır?

- API server webhook’a ulaşamazsa Pod yaratma işlemi takılır.
- TLS sertifikası geçersizse `x509: certificate signed by unknown authority` hatası çıkar.

### 🔹 Webhook’ları Listele

```bash
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
```

- Detay için `kubectl describe` kullan.

### 🔹 Webhook’u Geçici Olarak Devre Dışı Bırak

```bash
kubectl delete mutatingwebhookconfiguration <ADI>
```

- Acil çözüm için geçici bir yaklaşımdır.

## 🔐 RBAC Problemleri

### 🔹 Kullanıcı/POD Yetkisini Test Etme

```bash
kubectl auth can-i get pods --as=some-user -n <NAMESPACE>
```

- Erişim verilmemişse `no` sonucu döner.

### 🔹 Role ve RoleBinding Kontrolü

```bash
kubectl get role,rolebinding -n <NAMESPACE>
```

- Cluster-wide için `clusterrole` ve `clusterrolebinding` bakılır.

## 📝 Audit Log Problemleri (Kube-apiserver)

### 🔹 Audit Log Etkin mi?

- API sunucusu başlatılırken `--audit-log-path` ve `--audit-policy-file` parametreleri verilmiş mi?
- Dosya sisteminde audit log var mı?

```bash
cat /var/log/kubernetes/audit.log | grep <USER_ADI>
```

## 🛡️ Pod Güvenlik Ayarları

### 🔹 SecurityContext Kontrolü

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
```

- Container’ların root yetkisi olmaması için önemlidir.

### 🔹 PodSecurityPolicy (Deprecated) Yerine OPA/Gatekeeper

- Kubernetes v1.25 sonrası PSP kaldırıldı.
- Yerine Gatekeeper veya Kyverno gibi policy engineleri kullanılır.

## 🧬 etcd Sorunları

### 🔹 etcd Durumu ve Üyeler

```bash
ETCDCTL_API=3 etcdctl member list   --endpoints=https://127.0.0.1:2379   --cacert=/etc/kubernetes/pki/etcd/ca.crt   --cert=/etc/kubernetes/pki/etcd/server.crt   --key=/etc/kubernetes/pki/etcd/server.key
```

- Bir node erişilemiyorsa quorum kaybı olabilir (3 üyeli cluster için en az 2 erişilebilir olmalı).

### 🔹 etcd Snapshot Yedekten Geri Yüklemek

```bash
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db   --data-dir=/var/lib/etcd-from-backup
```

- Geri yükleme sonrası static pod manifest dosyası (`/etc/kubernetes/manifests/etcd.yaml`) bu dizine göre güncellenmeli.

## 📅 Sertifika Süreleri ve Yenileme

### 🔹 Sertifika Sürelerini Kontrol Etme

```bash
kubeadm certs check-expiration
```

- Kubernetes bileşenleri genellikle 1 yıl geçerli sertifikalarla başlatılır.

### 🔹 Tüm Sertifikaları Yenilemek

```bash
kubeadm certs renew all
systemctl restart kubelet
```

- Ardından kube-apiserver, controller, scheduler vs. restart olur.

## 🧹 `kubeadm reset` Sonrası Temizlik

### 🔹 `reset` Komutu

```bash
kubeadm reset
```

- Ardından iptables ve CNI kalıntıları temizlenmeli:

```bash
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
rm -rf /etc/cni/net.d
```

- Yeni `kubeadm init` öncesi bu adım önemlidir.

## ⚠️ Control Plane Failover (HA Ortamlar)

### 🔹 API Server Ulaşılmıyor

- Tüm control-plane node’ları down olduysa quorum kaybı olabilir.
- Load balancer varsa altındaki node’lar kontrol edilmeli.

### 🔹 etcd Yedeği Yoksa Tek Noda Dönme (Tehlikeli)

- Yalnızca bir `etcd` datası kalan node varsa `--force-new-cluster` ile yeniden başlatma yapılabilir.
- DİKKAT: Bu yöntem diğer üyelerle uyumsuzluk yaratır, en son çare olarak kullanılmalı.
