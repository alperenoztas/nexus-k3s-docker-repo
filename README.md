# K3s Üzerinde Nexus Kurulumu ve Docker Repo Ayarları

Kurulum yapacağımız işletim sistemi Ubuntu 24.04 server'dir. Sistem kaynak ayarları şu şekildedir: 

|  | Required | Used|
| ----------- | ----------- | ------------- |
| RAM | 2 GB | 4 GB|
| CPU | 2 Core | 4 Core|
| Storage | 30 GB | 50 GB |

> Kurulum yapmadan önce eğer statik olarak IP tanımlaması yapıyorsanız IP'nin subnet içinde kullanılmadığından emin olunması gerekir ping atarak teyit etmek doğrudur.

## K3s Kurulumu

Kurulum yapmak için verilen bu script yeterlidir:

```bash
curl -sfL https://get.k3s.io | sh -
```
> Eğer firewall'a sahipseniz firewall'da ayarlamalar yapmanız gerekebilir OS'ye göre gereken ayarlara [buradan](https://docs.k3s.io/installation/requirements) erişebilirsiniz.

Ardından sudo privilege kullanmadan erişebilmek ve kendi home dizinimize KUBECONFIG dosyasını yazdırmak için alttaki scriptleri kullanabilirsiniz.
```bash
export KUBECONFIG=~/.kube/config

mkdir ~/.kube 2> /dev/null

sudo k3s kubectl config view --raw > "$KUBECONFIG"

chmod 600 "$KUBECONFIG"
```

Container Network Interface, K3s kurulurken parametre olarak sağlanmadıysa Flannel kullanmak için aşağıdaki scripti kullanın." yerine "Eğer K3s kurulumu sırasında Container Network Interface (CNI) ayarlanmadıysa, Flannel kullanmak için aşağıdaki komutu çalıştırabilirsiniz.

```bash
#CNI kısmında kullanılmak üzere Flannel kullanılması
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

#Ardından controlplane node kısmının Ready olmasını bekleyiniz
kubectl get nodes
```

## Nexus Kurulum YAML dosyalarının uygulanması

Bu kısımda ilk önce Namespace oluşturarak Nexus'un izole edilmesi sağlanılacaktır.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: nexus
  labels:
    name: nexus
EOF
```

Uygulamamızın kalıcı bir şekilde çalışması için PersistentVolume ve PersistentVolumeClaim konfigürasyonlarını sağlayalım

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nexus-pv
  labels:
    pv: nexus-pv
    type: local
spec:
  capacity:
    storage: 15Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""
  hostPath:
    path: /nexus-data
  claimRef:
    name: nexus-pvc
    namespace: nexus
--- 
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-pvc
  namespace: nexus
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ""
  resources:
    requests:
      storage: 15Gi
  volumeName: nexus-pv
EOF
```

Nexus'un Uygulama Konfigürasyonları için ise Deployment'i uygulayalım.

```bash
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus
  namespace: nexus
  labels:
    app: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
    spec:
      containers:
        - name: nexus
          image: sonatype/nexus3:latest
          env:
          - name: MAX_HEAP
            value: "1000m"
          - name: MIN_HEAP
            value: "500m"
          ports:
            - containerPort: 8081
            - containerPort: 8082
            - containerPort: 8083
            - containerPort: 8084
          volumeMounts:
            - name: nexus-data
              mountPath: /nexus-data
      volumes:
        - name: nexus-data
          persistentVolumeClaim:
            claimName: nexus-pvc
EOF
```

Şimdi ise Service ile birlikte dışardan portlarımıza erişebilirliği sağlayalım.

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nexus-svc
  namespace: nexus
spec:
  selector:
    app: nexus
  type: NodePort
  ports:
    - name: nexus-web-ui
      port: 8081
      targetPort: 8081
      nodePort: 30881
    - name: nexus-docker-group
      port: 8082
      targetPort: 8082
      nodePort: 30882
    - name: nexus-docker-hosted
      port: 8083
      targetPort: 8083
      nodePort: 30883
    - name: nexus-docker-proxy
      port: 8084
      targetPort: 8084
      nodePort: 30884
EOF
```
Nexus Pod'umuzun __Running__ State'e ulaşmasını bekleylim

```bash
kubectl get pods -n nexus
```
Örnek Output: 
```bash
NAME                     READY   STATUS    RESTARTS       AGE
nexus-7f44c4d986-j2jfc   1/1     Running   1 (2d1h ago)   2d1h
```

## Nexus Uygulamasına Giriş

Kendi IP adresimiz ve Nexus'un UI kısmına erişmesi için sağlanan NodePort'u tarayıcınıza yazarak arayüze erişebiliriz:
> 192.168.1.105:30881 üzerinden ben erişiyorum nexus'un kurulu olduğu cihazda iseniz localhost:30881 üzerinden de erişebilirsiniz

Arayüz kısmında ilk defa giriş yapıldığı için şifremiz nexus container içerisinde /nexus/admin.password altında bu kısma erişmek için __-it__ flag'i ile interaktif bir şekilde erişebiliriz

```bash
kubectl exec -it -n nexus $(kubectl get pods -A -l "app=nexus" -o jsonpath="{.items[0].metadata.name}") -- /bin/bash
```

Ya da direkt kullanıma hazır bir şekilde almak için:

```bash
kubectl exec -n nexus $(kubectl get pods -A -l "app=nexus" -o jsonpath="{.items[0].metadata.name}") -- cat /nexus-data/admin.password
```

Bu adımdan sonra gelen şifreyle giriş sonrası şifrenizi değiştirebilirsiniz.

## Nexus Blob ve Repository Ayarları


### Blobs

İlk önce Nexus Reposun'da tutulacak docker image blob'ları(veya artifact'lar) için Blob Store oluşturmamıaz gerekiyor. Bunun için Nexus'a girdikten sonra __Server administration and configuration__ kısmında Blob Stores sekmesine tıklayarak gelebilirsiniz. Sonrasında __Create Blob Store__ butonuna basarak Type kısmında File seçiniz. Sonrasında buna isim olarak docker-hosted veriniz. Bu adımı docker-proxy isminde blob oluşturacak şekilde tekrarlayınız.

> Ayrıntılı repository ve blobs dokümanı için: [repository-management](https://help.sonatype.com/en/repository-management.html) ve [blobs-configurations](https://help.sonatype.com/en/configuring-blob-stores.html)


### Repository

Repository kısmında ise yine aynı sekme üzerinden erişim sağlanabilir buradan ise __Create repository__ butonuna tıklayarak repolarımızı oluşturabiliriz.

Bu kısımda biz kendi lokal imajlarımızı pushlayabilmek adına, uzak repodan lokalimizde olmayan imajlara ulaşmak için ve bunları gruplayabilmek için 3 adet repository tanımlayacağız bunlar:

- docker-proxy
- docker-hosted
- docker-group


Docker Hosted:

- name: docker-hosted
- type: hosted
- HTTP: 8083
- Allow anonymous docker pull: Enabled
- Enable Docker V1 API: enabled
- blob-store: docker-hosted

Docker Proxy:

- name: docker-proxy
- type: proxy
- HTTP: 8084
- Remote storage: https://registry-1.docker.io
- Docker Index: Use proxy registry
- Auto blocking enabled: Disabled
- Maximum component age: -1
- blob-store: docker-proxy

Docker Group:

- name: docker-group
- type: group
- HTTP: 8082
- Allow anonymous docker pull: Enabled
- Enable Docker v1 API: disabled
- blob-store: docker-hosted

## Image Push ve Pull için Containerd Konfigürasyonu ve Nexus Authentication

Bu kısımda Nexus(Private Registry) ile çalıştığımız için containerd config dosyası üzerinden bu ayarlamaları yapmanız gerekmektedir. Bu dizine erişmek için:

> Erişilemez ise dosya oluşturup devam edilmelidir.
- ``sudo cat /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl``
> NOT: __config.toml__ dosyası üzerinde değişiklilik yapmak doğru bir uygulama değil o yüzden __config.toml.tmpl__ üzerinden yapılmalıdır.

```ini
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."<your_ip>:30883"]
  endpoint = ["http://l<your_ip>:30883"]

[plugins."io.containerd.grpc.v1.cri".registry.configs."<your_ip>:30883".auth]
  username = "admin"
  password = "root" # Your nexus password

[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["http://<your_ip>:30882"]

[plugins."io.containerd.grpc.v1.cri".registry.mirrors."<your_ip>:8082"]
  endpoint = ["http://<your_ip>:30882"]

[plugins."io.containerd.grpc.v1.cri".registry.configs."<your_ip>:8082".auth]
  username = "admin"
  password = "root" # Your nexus password

```

Ayarların systemd tarafından tekrardan konfigüre edilmesi için:
```bash
sudo systemctl restart k3s
```

> Şuanki rehberimiz en kolay bir şekilde ayaklandırmayı gösterse bile şifrelerimizi bu şekilde tanımlamak yerine env olarak kullanabiliriz veya secret management kullanabiliriz.

## Image Pull ve Push İşlemleri Demo

Bu işlem için docker-hosted tarafına lokalimizde bulunan bir imajı push'lamak için:

```bash
sudo ctr image push --plain-http localhost:30883/library/nexus3:latest
```

İmajı çekmek için:

```bash
sudo ctr image pull localhost:30882/library/nginx:latest
```

## Proxy Yapısından Çekmeye Ekstra Örnek

Proxy yapılandırması, Docker Hub gibi uzak depolardan imajları indirirken Nexus'un bu isteği aracılık etmesine olanak tanır. Böylece sık kullanılan imajlar yerel olarak önbelleğe alınabilir ve daha hızlı erişim sağlanır.

Aşağıdaki komut, Nexus'ta tanımlı olan `docker-group` üzerinden Docker Hub'a ulaşır ve belirtilen imajı indirir:

```bash
ctr images pull localhost:30882/library/ubuntu:latest
```

Beklenen Sonuç:
``ctr: done``
___

İmajları kontrol etmek için ise:

```bash
ctr images list | grep ubuntu
```









