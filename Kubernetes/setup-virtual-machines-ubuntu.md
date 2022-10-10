## Konfiguracja VirtualBox

- Wyłącz Hyper-V (wykonaj z PowerShell)
```
bcdedit /set hypervisorlaunchtype off
```
- Wejdź w menu Plik->Ustawienia globalne->Sieć
- Stwórz nową sieć nat i nazwij ją Kubernetes
- Skonfiguruj stworzoną sieć
  - Adres sieci CIDR: 192.168.1.0/24
  - Reguły przekierowania portów:

| Nazwa  |  Protokół  |  IP hosta   | Port hosta |   IP gościa   | Port gościa |
|:------:|:----------:|:-----------:|:----------:|:-------------:|:-----------:|
| Rule 1 |    TCP     |  127.0.0.1  |   10022    | 192.168.1.100 |     22      |
| Rule 2 |    TCP     |  127.0.0.1  |   10023    | 192.168.1.10  |     22      |
| Rule 3 |    TCP     |  127.0.0.1  |   10024    | 192.168.1.11  |     22      |
| Rule 4 |    TCP     |  127.0.0.1  |   10025    | 192.168.1.12  |     22      |

## Przygotowanie maszyny Base

- Skonfiguruj nową maszynę
  - Nazwa: Base
  - Typ: Linux
  - Wersja: Ubuntu (64-bit)
  - Pamięć RAM: 2048 MB
  - Procesor: 2 CPU
  - Dysk twardy: 512 GB, dynamicznie przydzielany
  - Karta sieciowa: Sieć NAT o nazwie Kubernetes
- Zainstaluj Ubuntu Server LTS
  - Użytkownik: k8s
  - Hasło: k8s
  - Nazwa hosta: base
  - Zaznacz opcję: Install OpenSSH server
- Zaktualizuj system
```
sudo apt-get update
```
- Ustaw stały adres IP
```
sudo nano /etc/netplan/00-installer-config.yaml
```
```
network:
  ethernets:
    enp0s3:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.100/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
          addresses: [192.168.1.1]
  version: 2
```
```
sudo netplan apply
```
- Połącz się z maszyną wirtualną przez SSH
```
ssh -p 10022 k8s@localhost
```
- Zainstaluj Docker'a zgodnie z instrukcją [https://docs.docker.com/engine/install/ubuntu](https://docs.docker.com/engine/install/ubuntu)
```
sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
```
sudo mkdir -p /etc/apt/keyrings
```
```
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```
```
sudo usermod -aG docker $USER
```
- Zatrzymaj maszynę

## Przygotowanie klastra

- Sklonuj i skonfiguruj maszynę bazową
  - Nazwa: Master
  - Polityka adresów MAC: Wygeneruj nowe adresy MAC dla wszystkich kart sieciowych
- Zmień nazwę hosta
```
sudo hostnamectl set-hostname master
```
- Zmień adres IP
```
sudo nano /etc/netplan/00-installer-config.yaml
```
```
network:
  ethernets:
    ens3:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.10/24
      routes:
        - to: default
          via: 192.168.1.1
      nameservers:
          addresses: [192.168.1.1]
  version: 2
```
```
sudo netplan apply
```
- Wyłącz swap
```
sudo swapoff -a
```
```
sudo rm /swap.img
```
- Skonfiguruj /etc/hosts
```
sudo nano /etc/hosts
```
```
192.168.1.10 master    
192.168.1.11 node1    
192.168.1.12 node2    
192.168.1.100 admin
```
- Zrestartuj maszynę
- Zainstaluj narzędzie kubeadm zgodnie z instrukcją [https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm)
```
sudo apt-get update
```
```
sudo apt-get install -y apt-transport-https ca-certificates curl
```
```
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update
```
```
sudo apt-get install -y kubelet kubeadm kubectl
```
```
sudo apt-mark hold kubelet kubeadm kubectl
```
```
sudo rm /etc/containerd/config.toml
```
```
sudo systemctl restart containerd
```
- Zatrzymaj maszynę Master i sklonuj 2 razy, zmień nazwy i adresy nowych maszyn odpowiednio na:
  - node1: 192.168.1.11
  - node2: 192.168.1.12
- Stwórz klaster na maszynie Master
```
sudo kubeadm init
```
```
mkdir -p $HOME/.kube
```
```
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
```
```
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Pozwól na logowanie użytkownika root na maszynie Master
```
sudo nano /etc/ssh/sshd_config  (ustawić PermitRootLogin yes)
```
```
sudo /etc/init.d/ssh restart
```
- Ustaw hasło dla użytkownika root
```
sudo passwd root
```
- Dołącz każdy Nod1 i Node2 do klastra (użyj polecenia zwróconego przez kubeadm)
```
sudo kubeadm join 192.168.1.10:6443 --token o8wfxu.nkre9bljzbjfh9fv --discovery-token-ca-cert-hash sha256:dc27bdabb5ab24163ab8b5d67ce29fc5a4e550f86c71e4fe3bf377536fbeef15
```
## Przygotowanie maszyny administracyjnej

- Sklonuj i skonfiguruj maszynę bazową
    - Nazwa: Admin
    - Polityka adresów MAC: Wygeneruj nowe adresy MAC dla wszystkich kart sieciowych
- Zmień nazwę maszyny na: admin
- Zainstaluj kubectl zgodnie z instrukcją [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux)
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
- Przygotuj konfigurację potrzebną do połączenia z klastrem
```
mkdir ~/.kube
```
```
scp root@192.168.1.10:/etc/kubernetes/admin.conf ~/.kube/local
```
```
echo export KUBECONFIG=~/.kube/local >> ~/.bash_profile
```
- Dodaj ładowanie .bashrc w .bash_profile
```
if [ -f "$HOME/.bashrc" ]; then
   . "$HOME/.bashrc"
fi
```
- Skonfiguruj bash completion
```
echo "source <(kubectl completion bash)" >> ~/.bashrc
```
```
source .bash_profile
```
