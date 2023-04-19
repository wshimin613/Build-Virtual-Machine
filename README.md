# create-VM

## 硬體需求

硬體 | 建議配置 | 說明 |
--- | --- | --- |
處理器（CPU ）| > i5 或 i7 等級 | 目標是要完成虛擬化技術的建置，所以主機的 CPU 必須提供虛擬化的支援
記憶體（RAM） | > 12 GB        | 注意是 DDR3 還是 DDR4，最好有雙通道
硬碟         | 4顆             | 硬碟容量建議相同，建議配置4顆硬碟，搭配磁碟陣列運作

## 磁碟陣列

關於磁碟陣列分成軟體磁碟陣列、硬體磁碟陣列，磁碟陣列有許多種類型，一般常見的為 RAID 0、RAID 1、RAID 1+0、RAID 5、RAID 6  
而目前手邊沒有多餘的磁碟陣列卡，所以採用主機板的軟體磁碟陣列，並使用 RAID1+0 來建置（硬體磁碟陣列I/O能力比起軟體磁碟陣列好）  
![](https://i.imgur.com/pFO6mvt.png)

1. 開機按下 F2 進入 BIOS 後，選擇「進階」模式，選擇「SATA」按鈕將 SATA 的模式轉成 RAID (有 AHCI、RAID 與 IDE 等模式)
2. 按下 F10 儲存離開，之後在進入到 BIOS 之前，按下 ctrl + I 進入 RAID 頁面
3. 使用 RAID 10 機制處理

## 安裝作業系統
順序 | 介紹 |
--- | --- |
使用隨身碟開機，本章使用的是 Rocky Linux 9.0 | 開機時按 F8 選擇 UEFI 版本 |
硬碟規劃 | EFI System：250 M <br> /boot：1 G <br> swap：4 G（實體記憶體的一半） <br> /：給予剩下容量 |
Kdump | 關閉 |
安裝軟體 | 選擇最小安裝
帳戶設定 | 設定 root 密碼，以及一般用戶

## Host 環境設置

### 1. 基本軟體安裝
```bash
yum install  vim-enhanced bash-completion net-tools wget bind-utils -y
yum update -y
reboot
uname -r  #觀察系統核心
```
---

### 2. 時間校正
```bash
date
yum install epel-release -y
yum install chrony -y
vim /etc/chrony.conf
    server ntp.ksu.edu.tw iburst  #加入所需要的伺服器

systemctl start chronyd.service
systemctl enable chronyd.service
```
---

### 3. SELinux 設定  
SELinux 分別有兩種模式  
* Enforcing 是強制模式，在運作中會限制服務拒絕存取及記錄，如果要放行服務則要逐項做設定處理，相對的安全性會較高。
* Permissive 是寬容模式，SELinux會被啟用但不會實施安全性政策，而只會發出警告及記錄行動，通常在 Debug 時會以這兩個模式進行實驗測試。  
在這次的實作當中，由於使用的服務比較多，我們選擇 Permissive 模式來進行。
```bash
vim /etc/selinux/config
    #將 enforcing 改成 permissive
    SELINUX=permissive
#檢查
getenforce

#直接更換；0是 permissive，1是 enforcing
setenforce 0
```
---

### 4. 系統服務
由於是使用最小安裝，所以擁有的服務不多，但還是可以使用指令觀察目前系統所使用的服務項目。
* 讓 root 不能登入系統
* 關閉 DNS 反查功能
```bash
vim /etc/ssh/sshd_config
    #將 port 22 的註解拿掉
    #將 PermitRootLogin 的yes改成no
    #將 UserDNS no 的註解拿掉
    
systemctl restart sshd

#觀察系統服務：netstat
netstat -tlunp
# -t/ --tcp : 顯示 TCP 的連線狀況。
# -l/ --listening : 顯示監控中的伺服器的 Socket。
# -u/ --ucp : 顯示 UCP 的連線狀況。
# -n/ -numeric : 直接使用 IP Address，而不使用名稱伺服器。
# -P/ --programs : 顯示正在使用 Socket 的程式識別碼和程式名稱。
```
---

### 5. 防火牆建置
將內建 firewalld 關閉，改用 iptable.services
```bash
#停止、關閉 firewalld 服務
systemctl disable firewalld
systemctl stop firewalld

#安裝ipteables
yum install iptables-ser* -y

#啟動iptables服務
systemctl start iptables
systemctl enable iptables

#觀察iptables目前狀態
iptables-save
```
### `有關 iptables 腳本詳細內容，在上方供大家參考`

## 建立虛擬機
### 1. 主要虛擬化軟體與常用術語  

項目 | 介紹 |
---  | --- |
Host | 實體主機
Virtual Machine 簡稱 VM | 虛擬機器
Guest | VM 上面安裝的系統
Client	| 工作機
KVM | 整合到 Linux 核心，是最重要的虛擬化技術，可以虛擬出 CPU 的硬體
qemu | 虛擬出各項週邊設備，包括磁碟、網卡、USB、顯卡、音效等
libvirtd | 提供使用者一個管理 VM 的服務
virt-manager | 搭配 libvirtd 進行虛擬機器的管理
virsh | 終端機界面的管理指令
---

### 2. 安裝虛擬化環境
```bash
yum install virt-top libguestfs-tools
yum groupinstall "Virtualization Host"

systemctl start libvirtd
systemctl enable libvirtd
```
---

### 3. 建立虛擬磁碟
```bash
#使用qemu-img建立qcow2格式的磁碟檔案
#參考格式：qemu-img create -f qcow2 -o cluster_size=[512,1K,..2M] /vmdisk/your_image_filename.img sizeG

qemu-img create -f qcow2 rocky9.img 40G  # 製作一個 40G 的虛擬硬碟
```
---

### 4. 建立 XML 檔案  
* 在虛擬機器上安裝 Rocky Linux 的環境，因此要先下載該 ISO 檔案。
* 此外額外安裝virt-manager這個軟體，主要是協助虛擬機器部份管理操作。
```bash
# 前置作業
yum install virt-manager virt-install
wget http://ftp.ksu.edu.tw/FTP/Linux/rocky/9/isos/x86_64/Rocky-9.1-20221214.1-x86_64-dvd.iso  # 下載 ISO 檔

# 建立 XML 檔案
virt-install --name rocky9 \
	--cpu host --vcpus 4 --memory 4096 --memballoon virtio \
	--clock offset=utc \
	--controller virtio-scsi \
	--disk rocky9.img,cache=writeback,io=threads,device=disk,bus=virtio \
	--network network=qnet,model=virtio \
	--graphics spice,port=5930,listen=0.0.0.0,password=rocky9 \
	--cdrom Rocky-9.1-20221214.1-x86_64-dvd.iso \
	--video qxl \
	--dry-run --print-xml \
    > rocky9.xml
```
### `輸出成檔案後還有許多須更改的地方，因此提供修改後的檔案在上方供大家參考`
---

### 5. 設計虛擬網路橋接器  
* 虛擬機器連線到 Internet 有兩種方式  
  * 透過 Host 的 NAT 轉遞，取得 private IP 即可
  * 透過 Bridge 的功能，直接設定對外 IP 即可
```bash
# 利用 Linux 核心的 Bridge 功能
nmcli connection add type bridge con-name mybr0 ifname mybr3 ipv4.method manual ipv4.addresses 192.168.39.254/24
nmcli connection up
nmcli connection show
```
修改 xml 檔案內容
```bash
<interface type="bridge">
    <source bridge="mybr3"/>
    <mac address="52:54:00:5C:3F:B2"/>
    <model type="virtio"/>
</interface>
```
---

### 6. DHCP Server 設定  
為了讓 VM 未來可以自動取得 IP  
```bash
# Host 端

yum install dhcp -y
vim /etc/dhcp/dhcpd.conf
    subnet 192.168.39.0 netmask 255.255.255.0 {
        range 192.168.39.1 192.168.39.150;  # 分配 1 ~ 150 之間
        option routers 192.168.39.254;
        option domain-name "vir3.dic";
        option domain-name-servers 120.114.XXX.XXX;  # DNS 設定
    }
    
systemctl start dhcpd
systemctl enable dhcpd
```
虛擬機器也需要更改網路設定
```bash
# VM 端

nmcli connection modify eth0 ipv4.method auto ipv4.addresses '' ipv4.gateway '' ipv4.dns ''
nmcli connection up
```
---

### 7. 開啟虛擬機
```bash
virsh create rocky9.xml

virsh list
Id    名稱                         狀態
----------------------------------------------------
1     rocky9                      執行中
```
### `恭喜你，成功建立虛擬機!`
