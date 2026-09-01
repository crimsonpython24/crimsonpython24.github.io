# Debian 設置
應該架設自己的網站，不過目前挺懶o_O先放這裏將就一下，畢竟重設了幾次系統還是覺得得做個手冊。此設置建議避免使用apt以外的代碼庫來維持穩定性。

這手冊一部分是給我自己留着，所以中間會省略一些信息。請勿將此文獻當作唯一參考。

## 系統初始化
摘要：使用乙太（有線網路）並短暫關閉secure boot進行luks磁盤加密及初始Debian系統安裝。

參考來源：[DWArmstrong](https://www.dwarmstrong.org/minimal-debian/)、[CryptSetup](https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html)、[LUKS ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Cryptsetup_actions_specific_for_LUKS)。

### 前置作業
關閉secure boot。

### 安裝步䠫
選擇install：

![image](https://github.com/user-attachments/assets/62c39953-9563-49df-a2ad-6af8a3ef60b8)

如果「Detect network hardware」說hardware needs non-free firmware to operate（通常為ath10k/ath11k）選「no」跳過。在「Configure the network」頁面選擇`enp*`來使用乙太；以此電腦為例，`enp1s0f0`是瑞昱（Realtek）的Eth卡，而WiFi的`wlp2s0`介面則屬於高通（Qualcomm）。

建議開設root帳號並在需要時才登入（設置root時密碼不留空）。不要把使用者本身設置成root或加進sudoers。

到磁盤切割的時選擇手動：

![image](https://github.com/user-attachments/assets/14b93280-edf8-4ada-88b5-a153358a4539)

用標識的free space即可（vda通常為虛擬機磁盤，一般硬盤可能是`nvme0n*`）：

![image](https://github.com/user-attachments/assets/441495b2-9bf7-426f-861b-aaa41ba5f293)

這手冊的setup需要`/boot`（非加密），`/`（加密），和`/home`（加密）。`/boot`1GB-4GB，root（`/`）建議 32GB 以上（個人設置 160GB），剩下的丟`/home`。加密的選擇encryption。

選擇physical volume for encryption時使用磁盤精靈預設值即可。

![image](https://github.com/user-attachments/assets/51a08076-d6da-4920-bffd-82b55f34c4b2)

磁盤切割好時選擇Configure encrypted volumes並勾選兩個crypto的容量。等兩磁盤清理完（Erasing data on /dev/...）時，強烈建議使用相同密碼：

![image](https://github.com/user-attachments/assets/a1879756-f62b-4ca6-9ebb-1df39dce59aa)

密碼設置完成把mountpoint加入加密磁盤。設定好時應該長這樣（再次，`vda`是虛擬機的硬盤，如果是雙作業系統應該會寫`nvme0n*`）：

![image](https://github.com/user-attachments/assets/6c69e772-b0ec-40c5-becf-944455c2cd4f)

記得在新的crypto容量分別加上`/`和`/home`的mountpoint。此設置沒有swap磁盤，後面會設置`zramswap`；選「no」直接分割磁盤。

到這畫面只選擇standard system utilities（不用SSH；稍後安裝KDE是因為這步驟的KDE包含非必要的軟件）：

![image](https://github.com/user-attachments/assets/02318b09-1f57-45ec-b76e-c38f301b8d13)

### 軟件更新以及基本終端機設定
電腦重啓會看到以下畫面（如果有淺藍色背景代表安裝了`kde-plasma-desktop`；後面磁盤加密unmount如果黑屏大概只是SDDM被迫終止，故不建議在此及上段落直接裝kde）：

![image](https://github.com/user-attachments/assets/0ffef122-8d56-49c8-adfd-bfe325f2e1c3)

**記得移除含iso的外接媒體（如usb）**。輸入兩個加密磁盤（此手冊分順序root和home）的密碼並先用正常使用者登入。切換至root：

```sh
su - root
```

先開啓non-free及contrib的apt代碼庫（瑞昱及高通網卡基本都需要non-free韌體）：

```sh
nano /etc/apt/sources.list
```

每一行後面都確認包含所有的選項（例如Debian 11前的版本預設沒有`non-free-firmware`）：

```txt
deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
...
```

不建議使用backports及unstable。Unstable插件沒有經過完善的測試，所以可能跟現有的套件起衝突。例如撰寫時stable的`botan`是v2.19.3，unstable是2.19.5，如要修理dependency tree不保證能成功。

更改後跑`apt update`和`apt full-upgrade`。接下來確保韌體都有安裝。基於處理器安裝`intel-microcode`或`amd64-microcode`：

```sh
apt install amd64-microcode firmware-linux-nonfree firmware-atheros
```

### zram
Debian會建議開啓swap，但此手冊使用zram。zram會把磁盤的讀取/存儲從硬碟移進記憶體來避免不必要的硬盤操作及提升軟件讀取/存儲數據的速率。這是swap磁盤表的工作，但zram會把這些程序移進ram而不是硬碟。注意`zram`和`zswap`是不同的插件，而這教程使用前者。

```sh
swapoff --all
apt install zram-tools
zramswap stop
nano /etc/default/zramswap
```

更改以下參數（16GB以下建議使用25%，個人有32GB記憶體所以停留在50%）：

```txt
PERCENT=25 #範例
```

再執行：

```sh
grep swap /etc/fstab #應該空白
```

有輸出的話進入`/etc/fstab`把對應磁盤表comment掉。最後執行：

```sh
zramswap start
zramctl
# /dev/zram0 lz4 15.1G...
```

### 磁盤加密
磁盤加密有兩個部分：把root從luks2降級到luks1（Debian 12 bootloader只支持luks1，如使用luks2將無法載入加密後的`/boot`）。先處理root。執行`lsblk`：

```sh
lsblk
NAME                MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
sda                   8:0    1     0B  0 disk
zram0               252:0    0  15.1G  0 disk  [SWAP]
nvme0n1             259:0    0 476.9G  0 disk
├─nvme0n1p1         259:1    0   260M  0 part  /boot/efi
├─nvme0n1p2         259:2    0    16M  0 part
├─nvme0n1p3         259:3    0  93.9G  0 part
├─nvme0n1p4         259:4    0     2G  0 part
├─nvme0n1p5         259:5    0   3.7G  0 part
├─nvme0n1p6         259:6    0  74.5G  0 part
│ └─nvme0n1p6_crypt 253:0    0  74.5G  0 crypt /
└─nvme0n1p7         259:7    0 302.6G  0 part
  └─nvme0n1p7_crypt 253:1    0 302.6G  0 crypt /home
```

先操作root（`/dev/nvme0n1p6`）。確認此磁盤表是luks2且只有一個key slot（0）：

```sh
cryptsetup luksDump /dev/nvme0n1p6
LUKS header information
Version:       	2
[...]
Keyslots:
  0: luks2
```

接著重新啟動。重新啓動時GRUB頁面不要選擇Debian系統而按`e`進入booting parameters：

![image](https://github.com/user-attachments/assets/bbbecad3-e357-4e2a-ba52-36fb90b723fb)

以上僅是示意圖，不要加emergency。在linux那行尾端加上（quiet前面放一個空格）`break=mount`並按 `F10`載入initramfs介面。下次重啟時`break=mount`會自動移除。

這是會進入initramfs指令集而不是sh。此`nvme0n1p6`為root磁盤：

```sh
(initramfs) cryptsetup luksConvertKey --pbkdf pbkdf2 /dev/nvme0n1p6
(initramfs) cryptsetup convert --type luks1 /dev/nvme0n1p6
(initramfs) cryptsetup luksDump /dev/nvme0n1p6
```

最後一行`luksDump`應輸出`Version: 1`和`Key Slot 0: ENABLED`，其他處於DISABLED狀態。接著`reboot -f`重新啓動。

> 目前的狀態是GRUB不用密碼但`/`（`nvme0n1p6_crypt`）和`/home`（`nvme0n1p7_crypt`）各輸入一次密碼。如果GRUB在此步驟需要密碼，確認上一步加密的磁盤是`root`而不是`/boot`或`/home`。

接下來照常登入Debian（不是`e`選單）。登入一般使用者再換回root：

```sh
#先非sudoer使用者登入
#不是`su -`或是`su root`
su - root
```

執行：

```sh
mount -o remount,ro /boot
```

來避免複製中`/boot`被其他程序更改。切記初始安裝時`/boot`須爲ext4而非crypto。把`/boot`的內容複製到臨時目錄`/boot.tmp`再移除 `/boot`（`/boot/efi`通常是Windows的bootloader，不要更改）：

```sh
cp -axT /boot /boot.tmp
umount /boot/efi && umount /boot
rmdir /boot
mv -T /boot.tmp /boot
mount /boot/efi
```

如果以上指令沒問題，更改`/etc/fstab`移除`/boot`磁盤表紀錄：

```txt
#UUID=... /boot           ext4    defaults        0       2
```

這裏只是在目錄裡移除`\boot`而非移除磁盤表本身。在`/etc/default/grub`內加入

```txt
GRUB_ENABLE_CRYPTODISK=y
```

並執行

```sh
update-grub
grub-install /dev/nvme0n1
grep 'cryptodisk\|luks' /boot/grub/grub.cfg
```

其中`/dev/nvme0n1`在此例子中為root（`/dev/nvme0n1p6`）和boot（`/dev/nvme0n1p7`）的母磁盤。

最後一行應包含`insmod cryptodisk`和`insmod luks`來代表`/boot`已被加密。如`update-grub`輸出錯誤，確認安裝時的USB已經移除，不然系統會認定該裝置爲另一個作業系統而嘗試更新它的GRUB。沒問題就重新啓動，**但不要拔掉乙太線**。

> 重新啓動後的狀態應該是GRUB，`/`，和`/home`各需要一次密碼，總計三次輸入。GRUB密碼同root和home（開頭提到root和home建議使用同一密碼）。

### 使用者密鑰
目前要輸入三次密碼，但理想狀態是在GRUB輸入一次密碼，然後讓Debian自動載入root和home的加密磁盤（直接跳進使用者的登入提示）。GRUB密碼應等於root和home的密碼。從root開始生成使用者自己的密鑰：

```sh
su - root
dd bs=512 count=4 if=/dev/random of=/keyfile iflag=fullblock
chmod 600 /keyfile
cryptsetup luksAddKey /dev/nvme0n1p6 /keyfile
cryptsetup luksDump /dev/nvme0n1p6
```

`nvme0n1p6`是root的磁盤表。不要把keyfile加入boot或home（home後面會再加入另一個密鑰，但是要先生成root的密鑰，不然仍會多輸入一次密碼）。現在`luksDump`應顯示Version: 1，`Key Slot 0`和`Key Slot 1`處於ENABLED狀態，而其他為DISABLED。若密鑰無法使用，嘗試生成一個新的密鑰（cryptsetup會自己使用`Key Slot 2`但必須手動刪除逾期的`Key Slot 1`）。

接下來更改`/etc/crypttab`（由於`nvme0n1p6`的生成密鑰在slot 1，所以輸入key-slot=1）內的`nvme0n1p6_crypt`：

```txt
nvme0n1p6_crypt UUID=<a_long_string_of_characters> /keyfile luks,discard,key-slot=1
```

並更改`/etc/cryptsetup-initramfs/conf-hook`：

```txt
KEYFILE_PATTERN="/keyfile"
```

在`/etc/initramfs-tools/initramfs.conf`加入：

```txt
UMASK=0077
```

最後更新 initramfs：

```txt
update-initramfs -u -k all
```

到這一步時，root應該有自己的密鑰。使用以下指令檢查：

```sh
stat -L -c "%A  %n" /initrd.img
#-rw-------  /initrd.img
lsinitramfs /initrd.img | grep "^cryptroot/keyfiles"
#cryptroot/keyfiles
#cryptroot/keyfiles/nvme0n1p6_crypt.key
```

若keyfiles沒有正確生成，建議檢查/keyfiles有沒有在conf-hook裏更改及密鑰是加在root而非home（`/etc/crypttab`），並重新生成grub以及initramfs：

```sh
update-grub
grub-install /dev/nvme0n1
update-initramfs -u -k all
```

如果到這邊都沒問題，接下來生成home的密鑰。不用重啓電腦。執行：

```sh
dd bs=512 count=4 iflag=fullblock if=/dev/random of=/crypthome.key
chmod 600 /crypthome.key
```

把這個密鑰加入home（root已經有自己的密鑰）：

```sh
cryptsetup luksAddKey /dev/nvme0n1p7 /crypthome.key

#確認有成功加入密鑰：
cryptsetup luksDump /dev/nvme0n1p7
Version: 2
...
Keyslots:
  0: luks2
...
  1: luks2
...
```

由於沒有要把`/boot`載入`/home`，不用格式home到luks1。跟root一樣，現在home的`Key Slot 0`和`Key Slot 1`應各有一個密鑰。`Key Slot 0`是最初安裝Debian時設定的硬盤密碼，而Key Slot 1是讓Debian在使用者輸入GRUB密碼後自動解鎖的密鑰。再次更改 `/etc/crypttab`：

```sh
nvme0n1p7_crypt UUID=<a_long_string_of_characters> /crypthome.key luks,discard,key-slot=1
```

然後重啓電腦；確認乙太線仍然聯繫。如果安裝順利，只需在GRUB輸入一次密碼，Debian就會自動解鎖root和home並直接進入tty要求使用者登入。

## KDE 安裝
目前系統還在tty而沒有桌面介面。個人偏好KDE；之前用過Gnome但是感覺有點像Mac（偏好類似Windows的taskbar並無屏幕上方的navbar），且Gnome的半透明及視覺效果佔了不少記憶體。雖然KDE限制比較多（例如多語言輸入非常偏好`fcitx`而不是`dbus`，系統電池控制只限於`power-profiles-daemon`）但這是根據喜好。

值得一提：Gnome基於gdm而KDE是基於sddm的display manager。這邊不多做比較，但重點是Gnome的外觀及應用程序不一定能在KDE運行，反之亦然。同時個人感覺KDE的筆電支持，例如控制板以及觸控屏幕，比Gnome以及大多數的桌面環境好。

KDE使用的是 NetworkManager。安裝：

```sh
apt install network-manager network-manager-openvpn network-manager-config-connectivity-debian
```

這邊還無法ping。讓network-manager監管網路：

```sh
nano /etc/NetworkManager/NetworkManager.conf
```

```txt
[ifupdown]
managed=false #改成`managed=true`
```

由於安裝了`network-manager`（KDE的網路工具依賴它），必須把Debian本身的設定移除。開啓 `/etc/network/interfaces`並只留下lo：

```txt
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback
#把其他都移除
```

然後重啟network manager：

```sh
systemctl restart NetworkManager #執行完後等10秒
```

最後安裝 KDE：

```sh
apt install kde-plasma-desktop
```

>出現`IGN`狀態時終止安裝並跑`sudo apt autoremove`來移除已經安裝的插件。先確認乙太/Eth連線能用，再執行上面NetworkManager的步驟並跑`sudo systemctl disable NetworkManager.service`來重試連接網路。

到這裡仍不能ping，但別急。

安裝完KDE不要移除konqueror，因爲kde-baseapps依賴konqueror（截至bookworm這個依賴關係是hard depends），而kde-baseapps是kde-plasma-desktop的上游依賴之一。移除konqueror會讓apt的dependency tree出問題；即使沒有立即報錯也不代表系統穩定，沒必要爲了省一點空間而冒險。

最後再加splash screen（載入作業系統時登入前的特效，而不是tty的黑屏）：編輯`/etc/default/grub`，找到`GRUB_CMDLINE_LINUX_DEFAULT`並改成`splash`。執行`sudo update-grub`後重新啓動就能看到載入特效。

到這裡就可以重啓電腦並拔掉乙太線。首要任務是開啟secure boot，因為前置作業關閉了它。

## Secure Boot
每台電腦uefi都不同，以我的Thinkpad T14為例：進入grub菜單，選擇「UEFI Firmware Settings」進入Thinkpad的BIOS。接下來開啟secure boot和勾選「Allow Microsoft 3rd Party UEFI CA」，最後「save and exit」。由於Debian官方的系統有憑證，secure boot後應仍進入Debian bios選單而不是Windows boot manager。

啟用後回到Debian GNU/Linux，讓GRUB透過shim鏈接secure boot信任鏈（沒有這步，前面的UEFI設定不會有作用）：

```sh
apt install shim-signed grub-efi-amd64-signed
update-grub
mokutil --sb-state
#SecureBoot enabled
```

接下來可以盡情使用Debian~~看福瑞~~了。以下加裝只是推薦：

## 其他安裝
如無特別聲明，皆用`su - root`而非本地使用者。

### Xanmod
註冊Xanmod的PGP密鑰並加入代碼庫：

```sh
wget -qO - https://dl.xanmod.org/archive.key | sudo gpg --dearmor -vo /etc/apt/keyrings/xanmod-archive-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org $(lsb_release -sc) main" | sudo tee /etc/apt/sources.list.d/xanmod-release.list
#deb [signed-by=/etc/apt/keyrings/xanmod-archive-keyring.gpg] http://deb.xanmod.org trixie main
```

確認CPU的x86-64 psABI等級（E.g., Ryzen 5 Pro 6650U屬於Zen 3系列，用x64v3即可）：

```sh
apt update
apt install linux-xanmod-x64v3
```

apt install過程會自動跑update-initramfs和update-grub，不用手動再執行一次。

此代碼庫屬於第三方來源，跟本手冊開頭「避免使用apt以外代碼庫」的原則有出入。雖然加一獨立代碼庫（而非backports/unstable整體）風險比較可控，但仍歸類為選配。

Xanmod的核心沒有被Debian的簽名鏈信任，secure boot開啓時無法直接載入，需要自己生成MOK並簽名。先安裝簽名工具：

```sh
apt install sbsigntool
```

生成MOK密鑰：

```sh
mkdir -p /root/mok-keys && cd /root/mok-keys
openssl req -new -x509 -newkey rsa:2048 -keyout MOK.priv -outform DER -out MOK.der -nodes -days 36500 -subj "/CN=elrick-secureboot-key/"
chmod 600 MOK.priv
```

將MOK加入mokutil佇列並重啟。**MOK螢幕約5秒沒操作就會跳轉，必須盡快操作**。這裡會要求輸入一個一次性密碼，隨便打什麼都行：

```sh
mokutil --import MOK.der
mokutil --list-new
reboot
```

重啟後電腦會自動藍屏而不是進入一般的grub。選擇「Enroll MOK」接「Yes」並輸入上一步自定義的密碼，最後「reboot」。

這裏用Xanmod會跳出「bad shim signature」而不會使用此kernel；進入「Advanced options for Debian GNU/Linux」並用原本的Debian內核。登入並執行： 

```sh
mokutil --list-enrolled
#elrick-secureboot-key
```

將密鑰轉化成pem格式（sbsign只吃PEM格式的憑證，MOK.der本身仍保留給日後mokutil --import用）：

```sh
openssl x509 -in /root/mok-keys/MOK.der -inform DER -out /root/mok-keys/MOK.pem -outform PEM
```

創建簽名用的kernel postinst hook（每次新核心安裝/更新時會自動執行）：

```sh
nano /etc/kernel/postinst.d/zz-sign-kernel
```

```sh
#!/bin/sh
version="$1"
sbsign --key /root/mok-keys/MOK.priv --cert /root/mok-keys/MOK.pem \
    --output /boot/vmlinuz-$version.signed /boot/vmlinuz-$version
mv /boot/vmlinuz-$version.signed /boot/vmlinuz-$version
```

```sh
chmod +x /etc/kernel/postinst.d/zz-sign-kernel
```

保險起見手動補簽一次（往後apt升級核心會自動觸發，不用再手動跑）：

```sh
/etc/kernel/postinst.d/zz-sign-kernel 7.1.11-x64v3-xanmod1
```

確認：

```sh
sbverify --list /boot/vmlinuz-7.1.11-x64v3-xanmod1
lsinitramfs /boot/initrd.img-7.1.11-x64v3-xanmod1 | grep "^cryptroot/keyfiles"
```

重啟並在GRUB選擇Xanmod核心，確認能正常解鎖/並登入。登入後確認核心版本：

```sh
uname -r
#7.1.11-x64v3-xanmod1
```

到此步能跑`apt autoremove`來移除原始Linux內核。

### 防火牆

```sh
apt install ufw plasma-firewall
ufw enable
```

### Corectrl

```sh
apt install corectrl
```

確認polkit規則已裝好：

```sh
dpkg -L corectrl | grep polkit
```

在KDE的「系統設定/自動啟動」裡加Corectrl。Caveat：每次重啟後執行Corectrl前都會要求使用者密碼。先確認Corectrl實際的action ID：

```sh
cat /usr/share/polkit-1/actions/org.corectrl.helper*.policy
```

看`<action id="...">`確認action id（通常是`org.corectrl.helper.init`或類似名稱）。建立polkit規則檔：

```sh
nano /etc/polkit-1/rules.d/90-corectrl.rules
```

把`org.corectrl.helper.init`換成上一步查到的action id：

```js
polkit.addRule(function(action, subject) {
  if (action.id == "org.corectrl.helper.init" && subject.user == "elrick") {
    return polkit.Result.YES;
  }
});
```

這條規則只針對elrick帳號、只針對corectrl動作，不涉及sudo和sudoers。最後重啟polkit讓規則生效：

```sh
systemctl restart polkit
```

重啟後不用密碼會自動跳出corectrl。

### Fcitx5

```
apt install fcitx5 fcitx5-chinese-addons fcitx5-frontend-qt5 fcitx5-configtool
```

不要裝`kde-config-fcitx5`。輸入法在鍵盤上點右鍵更改選項較多。

### TLP

建議使用tlp來增加筆電電處續航，代價是無法從電池控制板調節省電/一般模式（必須移除ppd，平常沒用的話移除powertop）：

```sh
systemctl disable --now power-profiles-daemon
apt remove power-profiles-daemon powertop
apt install tlp tlp-rdw
systemctl enable --now tlp
```

選配安裝TLP-UI：

```sh
apt install python3-gi python3-gi-cairo gir1.2-gtk-3.0 libcairo2-dev gir1.2-girepository-2.0-dev pkg-config python3-pip
```

離開root返回一般使用者：

```sh
su - elrick
pip3 install --user tlp-ui --break-system-packages
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
tlpui #如果說cannot open display，開一個新的終端機視窗
```

沒有tlp-ui仍能用內建工具檢查tlp狀態：

```sh
tlp-stat -s
```

設置`/etc/tlp.conf`：

CPU

```
CPU_BOOST_ON_BAT=0
CPU_ENERGY_PERF_POLICY_ON_BAT=power
CPU_SCALING_GOVERNOR_ON_BAT=powersave
CPU_SCALING_MIN_FREQ_ON_AC=0
CPU_SCALING_MAX_FREQ_ON_AC=0
CPU_SCALING_MIN_FREQ_ON_BAT=0
CPU_SCALING_MAX_FREQ_ON_BAT=0
PLATFORM_PROFILE_ON_BAT=balanced #不要用low-power不然ryzenadj無法寫入
NMI_WATCHDOG=0
```

Audio Codec

```
SOUND_POWER_SAVE_ON_AC=0 #需要特別註明
SOUND_POWER_SAVE_ON_BAT=1
SOUND_POWER_SAVE_CONTROLLER=Y
```

PCIE

```
PCIE_ASPM_ON_BAT=powersupersave
RUNTIME_PM_ON_BAT=auto
```

電池充電值

```
START_CHARGE_THRESH_BAT0=75
STOP_CHARGE_THRESH_BAT0=80
```

Wi-Fi

```
DEVICES_TO_DISABLE_ON_BAT_NOT_IN_USE="bluetooth nfc wifi wwan"
WIFI_PWR_ON_BAT=on
```

GPU

```
RADEON_DPM_PERF_LEVEL_ON_BAT=low
```

最後確認使用schedutil：

/etc/default/grub

```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_pstate=active acpi.ec_no_wakeup=1 amdgpu.abmlevel=1 cpufreq.default_governor=schedutil"
#grub-update
```

### 全域DNS伺服器更改
創建全域DNS設置檔案：

```sh
nano /etc/NetworkManager/conf.d/00-global-dns.conf
```

```
[global-dns]
searches=
options=

[global-dns-domain-*]
servers=1.1.1.1,1.0.0.1
```

```sh
sudo systemctl restart NetworkManager
```

應看到：

```sh
nameserver 1.1.1.1
nameserver 1.0.0.1
```

### Ryzen 電源限制
#### 檢察核心鎖定
確認Lockdown LSM不會干預Ryzenadj的MSR寫入和PCI權限：

```sh
cat /sys/kernel/security/lockdown
```

如顯示任何除了`[none] integrity confidentiality`的提示，必須重新設置Secure boot（此系統未遇到著問題，所以只是揣測）。接著執行

```sh
ryzenadj -i
```

只要沒有報錯或是空白，即可繼續設置。

#### 開啟P-state
```sh
nano /etc/default/grub
```

加上amd_pstate：

```txt
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_pstate=active"
```

重新啟動電腦。測試：

```sh
update-grub
reboot
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_driver
#amd-pstate-epp
cat /sys/devices/system/cpu/amd_pstate/status
#active
```

#### 安裝Ryzenadj
```sh
apt install git cmake libpci-dev g++ make
git clone --recursive https://github.com/FlyGoat/RyzenAdj.git /root/RyzenAdj
cd /root/RyzenAdj
mkdir build && cd build
cmake ..
make
make install
```

確認msr會在重啟後執行：

```sh
echo msr > /etc/modules-load.d/ryzenadj.conf
modprobe msr
```

只要看到一串數字就沒問題。`no compatible ryzen_smu kernel module found, fallback to /dev/mem`代表沒有安裝`ryzen_smu`並使用`/dev/mem`的備用選項。

#### 載入k10temp
```sh
modprobe k10temp
echo k10temp > /etc/modules-load.d/k10temp.conf
```

執行`sensor`應看到：

```sh
sensors
#k10temp-pci-... 和 Tctl
```

#### 開啟背景程序
```sh
nano /etc/systemd/system/ryzenadj-tune.service
```

```txt
[Unit]
Description=Apply RyzenAdj power limits
After=multi-user.target tlp.service
Wants=tlp.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStartPre=/sbin/modprobe msr
ExecStart=/usr/local/bin/ryzenadj --stapm-limit=12000 --fast-limit=17000 --slow-limit=13000 --tctl-temp=85

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl enable --now ryzenadj-tune.service
systemctl status ryzenadj-tune.service
```

確認有「Successfully set...」即可。

#### 重載背景程序
```sh
nano /usr/lib/systemd/system-sleep/ryzenadj-resume
```

```sh
#!/bin/sh
case "$1" in
  post)
    systemctl restart ryzenadj-tune.service
    ;;
esac
```

```sh
chmod +x /usr/lib/systemd/system-sleep/ryzenadj-resume
echo balanced | sudo tee /sys/firmware/acpi/platform_profile
```

#### 確認程序
```sh
#重新啟動後
ryzenadj -i

#休息/resume
systemctl suspend

#醒來後
ryzenadj -i
journalctl -u ryzenadj-tune.service --since "5 min ago"
```

#### 測試（並調整參數）
```sh
stress-ng --cpu 0 --timeout 300s
watch -n1 sensors
```

## KDE 黑屏修復
如安裝了KDE重啓過電腦但仍還卡在tty，嘗試重新安裝sddm來修復 KDE：

```sh
apt install sddm
dpkg-reconfigure sddm
rm /etc/systemd/system/display-manager.service
ln -s /lib/systemd/system/sddm.service /etc/systemd/system/display-manager.service
systemctl enable sddm
systemctl start sddm
systemctl get-default
systemctl set-default graphical.target
```
