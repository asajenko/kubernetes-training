# Przygotowanie
- Zweryfikuj czy w bios/efi zostały włączone opcje wspierające wirtualizację
- Dla starszych wersji Windows, do prawidłowego działania VirtualBox, może być konieczne wyłączenie Hyper-V
```Powershell
bcdedit /set hypervisorlaunchtype off
```
- Na poziomie VirtualBox skonfiguruj sieć Nat z adresacją 192.168.1.0/24 i ustaw przekierowanie portu 22 na lokalną maszynę (Ustawienia globalne->Sieć->Sieci NAT)
## Konfiguracja maszyny bazowej
- Nazwa hosta: debian
- Hasło dla root: k8s
- Użytkownik sudo: k8s, hasło: k8s
- Adres IP: 192.168.1.100 (sieć typu Virtual NAT)
- Ram: 2048 Mb
- CPU: x2
- Zainstalowane pakiety
  - Server SSH
  - Podstawowe narzędzia systemowe
# Przygotowanie maszyny bazowej (po czystej instalacji systemu)
- Wyłącz cd/dvd jako źródło pakietów - usuń linię rozpoczynającą się od `deb cdrom`
```
nano /etc/apt/sources.list
```
- Zaktualizuj system i zainstaluj podstawowe narzędzia
```
apt update 
```
```
apt install sudo net-tools curl git
```
- Dodaj użytkownika k8s do grupy sudo
```
usermod -aG sudo k8s
```
- Wyłącz dhcp i ustaw stały adres IP
```
sudo nano /etc/network/interfaces
```
```
# iface enp0s3 inet dhcp
iface enp0s3 inet static
      address 192.168.1.100
      netmask 255.255.255.0
      gateway 192.168.1.1
```
- Zainstaluj Dockera https://docs.docker.com/engine/install/debian
```
sudo apt-get update
```
```
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```
```
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```
```
 echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
```
sudo apt-get update
```
```
sudo apt-get install docker-ce docker-ce-cli containerd.io
```
```
sudo usermod -aG docker $USER
```
- Zainstaluj docker-compose https://docs.docker.com/compose/install
```
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
```
sudo chmod +x /usr/local/bin/docker-compose
```
# Instalacja klastra
- Sklonuj maszynę bazową i nadaj jej nazwę master
```
sudo hostnamectl set-hostname master
```
- Wyłącz swap
```
swapoff -a
```
```
sed -i '/ swap / s/^/#/' /etc/fstab
```
- Zainstaluj kubeadm https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system
```
```
sudo apt-get update && sudo apt-get install -y apt-transport-https
```
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
```
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
```
```
sudo apt-get update
```
```
sudo apt-get install -y apt-transport-https ca-certificates
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
- Ustaw adress IP
```
sudo nano /etc/network/interfaces
```
```
# iface enp0s3 inet dhcp
iface enp0s3 inet static
      address 192.168.1.10
      netmask 255.255.255.0
      gateway 192.168.1.1
```
- Zdefiniuj adresy pozostałych maszyn
```
sudo nano /etc/hosts
```
```
192.168.1.10 master    
192.168.1.11 node1    
192.168.1.12 node2    
192.168.1.100 admin
```
- Sklonuj maszynę master dwa razy i ustaw nazwy oraz adresy IP dla node1 i node2
- Zainicjuj klaster na maszynie master
```
sudo kubeadm init
```
- Dołącz pozostałe maszyny (node1, node2) postępując zgodnie z wyświetlaną instrukcją
- Zainstaluj warstwę sieciową
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```
- Pozwól na logowanie użytkownika root na maszynie master
```
nano /etc/ssh/sshd_config  (ustawić PermitRootLogin yes)
```
```
/etc/init.d/ssh restart
```
- Sklonuj maszynę bazową i ustaw nazwę i adres IP dla admin
- Zainstaluj kubectl https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```
```
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```
- Skopiuj konfigurację z maszyny master
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
# Inne
- Wyświetlenie wszystkich logów na Master
```
journalctl -u kubelet --no-pager|less
```
- Opcjonalna wymiana certyfikatu jeśli nie użyliśmy --apiserver-cert-extra-sans
```
rm /etc/kubernetes/pki/apiserver.*
kubeadm phase certs all --apiserver-advertise-address=0.0.0.0 --apiserver-cert-extra-sans=x.x.x.x,x.x.x.x
docker rm -f `docker ps -q -f 'name=k8s_kube-apiserver*'`
systemctl restart kubelet
```
- Opcjonalnie wyświetlenie tokenów
```
kubeadm token list
```
