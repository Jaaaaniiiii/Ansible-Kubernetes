# Ansible-Kubernetes

## Hands-On Lab 1: Ansible Web Server Deployment
### Tujuan
Menggunakan Ansible untuk mengotomatisasi deployment Nginx web server di Ubuntu dengan konten dinamis yang menampilkan hostname dan IP address.

---
# 🎯 Prasyarat & Environment Setup

| Role         | Hostname   | IP Address     | Purpose               |
| ------------ | ---------- | -------------- | --------------------- |
| Control Node | cloud      | 192.168.56.105 | Menjalankan Ansible   |
| Managed Node | cloudclone | 192.168.56.106 | Target server (Nginx) |

Pastikan kedua VM dapat saling ping:

```bash
ping 192.168.56.106
```

---

# 1. Install Ansible di Control Node

```bash
# Update package list
sudo apt update

# Install Ansible dan SSH client/server
sudo apt install -y ansible openssh-client openssh-server

# Verifikasi instalasi
ansible --version
```
<img width="696" height="407" alt="image" src="https://github.com/user-attachments/assets/d5891033-cb91-436d-a9fb-59658ececf6b" />


---

# 2. Konfigurasi SSH Key-Based Authentication

```bash
# Generate SSH key pair (di control node)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa_ansible -N ""

# Copy public key ke managed node
ssh-copy-id -i ~/.ssh/id_rsa_ansible.pub cloud@192.168.56.106

# Jika SSH belum aktif di target, install dulu:
sudo apt install -y openssh-server
sudo systemctl enable --now ssh

# Test SSH access tanpa password
ssh -i ~/.ssh/id_rsa_ansible cloud@192.168.56.106
```
<img width="501" height="219" alt="image" src="https://github.com/user-attachments/assets/9d3c4f21-74bd-41e8-bb88-3d1663ce71c7" />
<img width="709" height="176" alt="image" src="https://github.com/user-attachments/assets/b99c562d-5475-48c6-b92c-8b79e5963e96" />
<img width="707" height="366" alt="image" src="https://github.com/user-attachments/assets/9b7ff8d1-6d99-41a4-9873-9c1a89d6c18b" />
<img width="429" height="337" alt="image" src="https://github.com/user-attachments/assets/d289dd02-9620-430d-9fd4-5dcb93b98a46" />

---

# 3. Buat Struktur Project Ansible

```bash
mkdir -p ~/ansible-project
cd ~/ansible-project

# Struktur direktori:
~/ansible-project/
├── inventory
├── site.yml
└── index.html.j2
```
<img width="353" height="27" alt="image" src="https://github.com/user-attachments/assets/77c139ef-88de-46c6-8a2a-255851791cbf" />

---

# 4. Buat Inventory File

```bash
cat > inventory << 'EOF'
[webservers]
192.168.56.105 ansible_user=cloud ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
192.168.56.106 ansible_user=cloud ansible_ssh_private_key_file=~/.ssh/id_rsa_ansible
EOF

# Verifikasi inventory:
ansible-inventory -i ./inventory --list
```
<img width="529" height="520" alt="image" src="https://github.com/user-attachments/assets/ec898c04-dfa5-49c9-8ffe-6daa95983950" />

---

# 5. Test Konektivitas

```bash
# Ping test ke semua webservers
ansible -i ./inventory webservers -m ping

# Expected output:
192.168.56.106 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```
<img width="549" height="165" alt="image" src="https://github.com/user-attachments/assets/6725b156-a382-44ac-b378-16e1efbffd67" />

---

# 6. Buat Main Playbook (`site.yml`)

```bash
cat > site.yml << 'EOF'
---
- name: Deploy nginx and simple web page on webservers
  hosts: webservers
  become: yes
  gather_facts: yes

  tasks:
    - name: Update apt cache
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Ensure nginx is installed
      ansible.builtin.apt:
        name: nginx
        state: present

    - name: Ensure /var/www/html exists
      ansible.builtin.file:
        path: /var/www/html
        state: directory
        mode: '0755'
        owner: www-data
        group: www-data

    - name: Deploy index.html showing hostname and IPs
      ansible.builtin.template:
        src: index.html.j2
        dest: /var/www/html/index.html
        mode: '0644'
        owner: www-data
        group: www-data

    - name: Ensure nginx is started and enabled
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: yes
EOF
```
<img width="409" height="544" alt="image" src="https://github.com/user-attachments/assets/5a50b739-5258-4935-81bf-55e1f0fc5e9c" />

## Penjelasan Playbook

* `become: yes` → Jalankan dengan sudo privileges
* `gather_facts: yes` → Kumpulkan info sistem (hostname, IP, dll)
* `ansible.builtin.apt` → Module untuk package management
* `ansible.builtin.template` → Deploy file dari template Jinja2
* `ansible.builtin.service` → Manage systemd services

---

# 7. Buat HTML Template (Jinja2)

```bash
cat > index.html.j2 << 'EOF'
<html>
<head><title>Deployed by Ansible</title></head>
<body>
  <h1>Hello from {{ ansible_hostname }}!</h1>
  <p>IP addresses:</p>
  <ul>
    {%- if ansible_default_ipv4 is defined and ansible_default_ipv4.address is defined %}
    <li>{{ ansible_default_ipv4.address }}</li>
    {%- else %}
    {%- for iface in ansible_facts.interfaces | sort %}
    {%- set key = 'ansible_' + iface %}
    {%- if (key in ansible_facts) and (ansible_facts[key].ipv4 is defined) %}
    <li>{{ iface }}: {{ ansible_facts[key].ipv4.address }}</li>
    {%- endif %}
    {%- endfor %}
    {%- endif %}
  </ul>
  <p>Deployed on {{ ansible_date_time.date }} {{ ansible_date_time.time }}</p>
</body>
</html>
EOF
```
<img width="566" height="295" alt="image" src="https://github.com/user-attachments/assets/abb2919b-3aa4-4720-b738-8d5ac1da2996" />

## Variabel Jinja2 yang Digunakan

* `{{ ansible_hostname }}` → Nama host target
* `{{ ansible_default_ipv4.address }}` → IP address default
* `{{ ansible_facts.interfaces }}` → List network interfaces
* `{{ ansible_date_time.date }}` → Tanggal deployment

---

# 8. Jalankan Playbook

```bash
# Jalankan playbook
ansible-playbook -i ./inventory site.yml

# Jika diminta sudo password
ansible-playbook -i ./inventory site.yml -K

#Expected output:
PLAY [Deploy nginx and simple web page on webservers]

TASK [Gathering Facts]
ok: [192.168.56.106]

TASK [Update apt cache]
changed: [192.168.56.106]

TASK [Ensure nginx is installed]
changed: [192.168.56.106]

TASK [Ensure /var/www/html exists]
ok: [192.168.56.106]

TASK [Deploy index.html]
changed: [192.168.56.106]

TASK [Ensure nginx is started]
ok: [192.168.56.106]

PLAY RECAP
192.168.56.106 : ok=6 changed=3 unreachable=0 failed=0
```
<img width="1080" height="588" alt="image" src="https://github.com/user-attachments/assets/fdd8f851-ac0e-4945-baf9-97e578af4c4a" />

---

# 9. Verifikasi Deployment

```bash
# Cek status nginx
systemctl status nginx

# Test dari control node menggunakan curl
curl http://192.168.56.106
```
<img width="395" height="140" alt="image" src="https://github.com/user-attachments/assets/9ab23462-32ff-4183-99a3-3101352795ad" />
<img width="309" height="134" alt="image" src="https://github.com/user-attachments/assets/040c8a2a-5f55-4496-919d-27a116619ef3" />

Atau buka browser:

```bash
http://192.168.56.106
```
<img width="339" height="118" alt="image" src="https://github.com/user-attachments/assets/6e542723-90d6-4f84-b133-c4e21dbe2b4c" />
<img width="258" height="119" alt="image" src="https://github.com/user-attachments/assets/d83a8e89-95b6-4832-be42-c653c3255240" />


## Expected Result

Browser akan menampilkan:

* Judul: `Deployed by Ansible`
* Heading: `Hello from cloudclone!`
* List IP addresses
* Timestamp deployment

---

# 📋 Checklist Lab Ansible

- [x] VM Control Node dan Managed Node saling terhubung
- [x] Ansible terinstall di Control Node
- [x] SSH key-based authentication berfungsi tanpa password
- [x] Inventory file terkonfigurasi dengan benar
- [x] Playbook berhasil dijalankan tanpa error
- [x] Nginx berjalan dan enabled di managed node
- [x] Halaman web menampilkan hostname dan IP dinamis
- [x] Template Jinja2 berhasil dirender dengan variabel Ansible facts

---

# ☸️ Hands-On Lab 2: Kubernetes Basics

## Tujuan

Memahami konsep dasar Kubernetes dengan membuat Pod, Deployment, dan Service menggunakan Minikube atau cluster lokal.

---

# 🎯 Prasyarat

* Minikube terinstall
* Kubectl terinstall
* Docker atau container runtime tersedia

Install dependency:

```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# Install kubectl
sudo snap install kubectl --classic
```

---

# 1. Start Minikube Cluster

```bash
# Start minikube dengan driver docker
minikube start --driver=docker

# Verifikasi status
minikube status

# Cek nodes
kubectl get nodes
```

---

# 2. Buat Pod Manifest

```bash
cat > pod-nginx.yml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: web
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF
```

---

# 3. Deploy Pod

```bash
# Apply manifest
kubectl apply -f pod-nginx.yml

# Lihat pod yang berjalan
kubectl get pods -o wide

# Detail pod
kubectl describe pod nginx-pod

# Logs pod
kubectl logs nginx-pod
```

---

# 4. Buat Deployment (Best Practice)

```bash
cat > deployment-nginx.yml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "100m"
          limits:
            memory: "128Mi"
            cpu: "200m"
EOF
```

## Deployment vs Pod

Deployment mengelola ReplicaSet yang mengelola Pods. Jika Pod mati, Deployment akan otomatis membuat Pod baru untuk memenuhi `replicas: 3`.

---

# 5. Deploy dan Scale

```bash
# Apply deployment
kubectl apply -f deployment-nginx.yml

# Cek deployment dan pods
kubectl get deployment
kubectl get pods

# Scale ke 5 replicas
kubectl scale deployment nginx-deployment --replicas=5

# Cek lagi
kubectl get pods
```

---

# 6. Buat Service (Exposing Pods)

```bash
cat > service-nginx.yml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: NodePort
EOF
```

Apply service:

```bash
kubectl apply -f service-nginx.yml
```

Cek service:

```bash
kubectl get service nginx-service
```

Akses service:

```bash
# Mendapatkan URL
minikube service nginx-service --url

# Membuka browser otomatis
minikube service nginx-service
```

---

# 7. Self-Healing Demo

```bash
# Lihat pods
kubectl get pods

# Hapus salah satu pod
kubectl delete pod nginx-deployment-xxx-yyy

# Observe auto-healing
kubectl get pods -w
```

Tekan `Ctrl + C` untuk berhenti watch.

## Self-Healing

Kubernetes mendeteksi bahwa jumlah pod berkurang dari desired state. Controller Manager otomatis membuat pod baru untuk mengembalikan state yang diinginkan.

---

# 8. Rolling Update

```bash
# Update image
kubectl set image deployment/nginx-deployment nginx=nginx:1.26

# Monitor rolling update
kubectl rollout status deployment/nginx-deployment

# Cek history
kubectl rollout history deployment/nginx-deployment

# Rollback jika ada masalah
kubectl rollout undo deployment/nginx-deployment
```

---

# 9. Cleanup

```bash
# Hapus resources
kubectl delete -f pod-nginx.yml
kubectl delete -f deployment-nginx.yml
kubectl delete -f service-nginx.yml
```

Atau sekaligus:

```bash
kubectl delete all --all
```

Stop Minikube:

```bash
minikube stop
```

---

# 📋 Checklist Lab Kubernetes

* Minikube cluster berjalan tanpa error
* Pod nginx berhasil dibuat dan running
* Deployment dengan 3 replicas berjalan
* Scaling dari 3 ke 5 replicas berhasil
* Service NodePort berhasil expose pods
* Self-healing bekerja saat pod dihapus
* Rolling update ke image baru berhasil
* Rollback ke versi sebelumnya berfungsi

---

# 🧠 Konsep Kunci yang Dipelajari

## 📦 Pod

Unit terkecil yang berisi satu atau lebih container yang share network dan storage.

## 📋 Deployment

Mengelola ReplicaSet untuk memastikan jumlah pod sesuai desired state.

## 🔌 Service

Abstraksi yang mendefinisikan logical set of pods dan policy untuk mengaksesnya.

## 🎯 Desired State

Kubernetes terus-menerus membandingkan actual state dengan desired state dan melakukan koreksi.

---

# 🎓 Ringkasan & Evaluasi

## Perbedaan Automation vs Orchestration

### Automation

Script yang menjalankan tugas individual seperti install nginx atau copy file.

### Orchestration

Sistem yang mengkoordinasikan banyak automation untuk mencapai system state yang diinginkan seperti deploy multi replicas, load balancing, auto-healing, dan scaling.

---

# Kapan Menggunakan Ansible vs Kubernetes?

| Skenario                                          | Tools               | Alasan                         |
| ------------------------------------------------- | ------------------- | ------------------------------ |
| Provision VM, install software, konfigurasi OS    | Ansible             | Agentless, idempotent, simple  |
| Manage container lifecycle, scaling, self-healing | Kubernetes          | Container orchestration native |
| Deploy infrastructure di cloud                    | Terraform + Ansible | Infrastructure as Code         |
| CI/CD pipeline automation                         | Jenkins / GitLab CI | Build, test, deploy automation |

---

# ❓ Pertanyaan Diskusi

1. Mengapa cloud provider membutuhkan automation tools yang lebih sophisticated dibanding individual users?
2. Apa risiko jika hanya menggunakan script bash untuk manage 1000+ containers tanpa orchestrator?
3. Bagaimana Kubernetes menangani situasi ketika sebuah worker node mati total?
4. Jelaskan peran etcd dalam menjaga consistency cluster state!
5. Mengapa Pod menggunakan IP address yang berbeda-beda, dan bagaimana Service mengatasi hal ini?
