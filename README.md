# Debian 設置
應該架設自己的網站，不過目前有點懶 o_O 先放在這裏將就一下（畢竟把 OS 重設了幾次覺得可能得做一個類似的手冊）

我沒有太多 Linux 的經驗，如果有問題請勿噴太慘。先說明一下不是每個 Ubuntu 的設定都可以帶入 Debian；這個設置也盡可能避免使用 apt 以外的代碼庫來維持穩定性。

這篇手冊一部分是給我自己留着，所以中間可能會省略一些信息。

## 警告/Warning/Advertencia
目前這篇文章還未檢查，請勿將此文獻當作唯一參考。

This article is not reviewed yet. Please do not cite this as the only resource.

Este artículo no ha sido revisado aún. No lo utilice como única referencia, por favor.

## 系統初始化
摘要：使用 luks 磁盤加密，xanmod kernel,以及使用 secure boot 設置

參考來源：[DWArmstrong](https://www.dwarmstrong.org/minimal-debian/)，更多參考：[LinuxReviews](https://linuxreviews.org/Zram)，[CryptSetup](https://cryptsetup-team.pages.debian.net/cryptsetup/encrypted-boot.html)，[LUKS ArchWiki](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption#Cryptsetup_actions_specific_for_LUKS)

### 安裝步䠫
安裝的時候，選擇 install：
![image](https://github.com/user-attachments/assets/62c39953-9563-49df-a2ad-6af8a3ef60b8)

到磁盤切割的時候，選擇手動：
![image](https://github.com/user-attachments/assets/14b93280-edf8-4ada-88b5-a153358a4539)

直接用標識的 free space 即可（最簡單的辦法是直接從 Windows 把舊的 partition 清空）：
![image](https://github.com/user-attachments/assets/441495b2-9bf7-426f-861b-aaa41ba5f293)

需要 `/boot`（非加密），一個 `/`（加密），以及一個 `/home`（加密）。`/boot` 1GB-4GB, root（`/`）建議 16GB 以上（個人設置 64GB），剩下的就丟 `/home`。以上需要加密的選擇 encryption：
![image](https://github.com/user-attachments/assets/51a08076-d6da-4920-bffd-82b55f34c4b2)

磁盤切割好的時候選擇 configure ecrypted volumes 並勾選兩個 crypto 的容量：
![image](https://github.com/user-attachments/assets/a1879756-f62b-4ca6-9ebb-1df39dce59aa)

密碼設置完成時記得把 mountpoint 加入加密磁盤。設定好時應該長這樣（其中 `vda` 是虛擬機裏面的 virtual disk，如果是雙作業系統應該會寫 `nvme0n*`）：
![image](https://github.com/user-attachments/assets/6c69e772-b0ec-40c5-becf-944455c2cd4f)

這個設置並沒有使用 swap，後面會設置 `zramswap`。如果有另外的 swap 磁盤表建議也把此空間加密。

到這個畫面的時候只選擇 standard system utilities（Debian 預設會有 Gnome；這個手冊會安裝 KDE，但是如果在這一步選擇 KDE 的話會安裝一些不必要的軟件，先從 tty/default shell 開始就好）：
![image](https://github.com/user-attachments/assets/02318b09-1f57-45ec-b76e-c38f301b8d13)

### 軟件更新以及基本終端機設定
如果電腦成功重啓會看到以下的畫面（如果是一個淺藍色背景的屏幕，那是安裝了 `kde-plasma-desktop`；由於後面磁盤加密會 unmount,電腦可能會黑屏，不過大概率只是 SDDM 被迫終止而已，`ctrl-alt-f2` 可以切換回 tty）：
![image](https://github.com/user-attachments/assets/0ffef122-8d56-49c8-adfd-bfe325f2e1c3)

建議開啓 non-free 以及 contrib 的 apt 代碼庫（瑞昱以及驍龍的網卡基本都是 non-free 的韌體，如果有 nvidia 的裝置也得使用非開源的軟件）：
```sh
$ sudo nano /etc/apt/sources.list
```
每一行後面都確認包含所有的選項，例如（Debian 12 以前的版本可能不會有 `non-free-firmware`）：
```
deb http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
deb-src http://deb.debian.org/debian/ bookworm main contrib non-free non-free-firmware
...
```
這邊不建議使用 backports 或是 unstable。個人在安裝 corectrl 時有遇到這個問題：backports 有 v1.3 的舊版本，切換到 unstable 可以下載 v1.4.2。要注意的是 unstable 的插件沒有經過完善的測試，所以可能跟現有的 dependencies 起衝突。例如 stable 的 botan 是 v2.19.3，unstable/sid 會是 2.19.5，這樣 apt 就必須修理 dependency tree，但不保證能成功。

同時，Debian 12 的 backports 理論上都可以用 stable 的 apt 代碼庫運行，但是基於穩定性還是不建議安裝（backports 是把 unstable 的軟件更改到可以用 stable 的代碼運行，但是沒有保證會是最新的版本或是能完全運行）。

更改完之後跑 `sudo apt update` 和 `sudo apt full-upgrade`。這邊建議安裝以下三個指令：
```sh
$ sudo apt install command-not-found apt-file
$ sudo apt-file update && sudo update-command-not-found
$ sudo apt install plocate && sudo /etc/cron.daily/plocate
```
`command-not-found` 會建議未安裝的指令。例如這時跑 `valgrind` 大概率不會成功（這是 c/c++ 檢測記憶體流失的工具程式，但是不含在 Debian 的初始安裝裏面）。這時 command-not-found 就會提醒使用 apt 安裝 valgrind。`apt-file` 以及 `plocate` 個人比較少用到，但是可能會對索引 apt 插件以及使用者檔案有幫助。

接下來確保韌體都有安裝完成（手動安裝應該會自己偵測處理器架構並且安裝韌體，但是建議檢查一下）。基於處理器安裝 `intel-microcode` 或 `amd64-microcode`：
```sh
$ sudo apt install amd64-microcode
```

### zram
手動安裝時 Debian 會建議開啓 swap，但是個人建議使用 zram 。zram 會把磁盤的讀取/存儲從硬碟移進記憶體，這樣可以避免不必要的硬盤讀取（ram 只有應用程序有需要的時候才會寫入，所以大部分時間都有空位），也可以提升軟件讀取/存儲數據的速率。這個是 swap 磁盤表的工作，但是 zram 會把這些程序移進 ram 而不是硬碟。注意 `zram` 和 `zswap` 是不同的插件，而這邊使用前者。
```sh
$ sudo swapoff --all
$ sudo apt install zram-tools
$ sudo zramswap stop
$ sudo nano /etc/default/zramswap
```
在這個檔案裏面，增加以下指令（16GB 以下建議使用 25%，個人有 32GB 的記憶體所以更改爲 50%）：
```
PERCENT=25
PRIORITY=100
```
再執行：
```sh
$ grep swap /etc/fstab
/dev/mapper/vgmint-swap_1 none            swap
```
有輸出的話進入 `/etc/fstab` 把對應的磁盤表 comment 掉，沒輸出的話就沒有問題。最後再執行：
```
$ sudo zramswap start
$ sudo zramctl
```

### 磁盤加密
磁盤加密有兩個部分：把 root 從 luks2 降級到 luks1（Debian 的 bootloader 只支持 luks1，如果使用 luks2 無法載入加密後的 `/boot`），然後加入使用者密鑰來避免重新輸入密碼。這邊先處理 root 的磁盤表。先執行 `lsblk`：
```sh
$ lsblk
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
要改成 luks1 的是 `nvme0n1p6`，也就是 `/` 的 mountpoint。同時確認這個磁盤表是 luks2 並且只有一個 key slot（0）：
```sh
$ sudo cryptsetup luksDump /dev/nvme0n1p6
LUKS header information
Version:       	2
[...]
Keyslots:
  0: luks2
```
這時就可以重啓電腦並把 root 格式化成 luks1，但沒辦法在 Debian 運行的時候更改。重新啓動時在 GRUB 的頁面選擇 Debian 的系統，但是按 `e`（而不是 `enter`）進入 booting parameters：
![image](https://github.com/user-attachments/assets/bbbecad3-e357-4e2a-ba52-36fb90b723fb)

以上只是示意圖，不用加 emergency。接下來在 linux 那行後面加上（break 前面放一個空格）`break=mount`，並按 `F10` 載入 initramfs。這時確認在 initramfs 中而不是使用者的指令集（登入使用者的指令集會要求名稱以及密碼，但是 initramfs 可以直接使用。initramfs 的 `cryptsetup` 也不用 `sudo`）：
```sh
(initramfs) cryptsetup luksConvertKey --pbkdf pbkdf2 /dev/nvme0n1p6
(initramfs) cryptsetup convert --type luks1 /dev/nvme0n1p6
(initramfs) cryptsetup luksDump /dev/nvme0n1p6
```
最後一行的 `luksDump` 應該輸出 `Version: 1` 和 `Key Slot 0: ENABLED`。Key Slot 1 到 7 應處於 DISABLED 狀態。接着按 `CTRL-ALT-DELETE` 重新啓動。

> 目前的狀態是 GRUB 不用密碼，但是 `/`（`nvme0n1p6_crypt`）和 `/home`（`nvme0n1p7_crypt`）分別各輸入一次密碼。如果 GRUB 要密碼或是 home/root 其中一個不用密碼，確認加密的 partition 是 `root` 而不是 `/boot` 或 `/home`。

接下來進去 Debian（不是 initramfs）並照常登入。執行：
```sh
$ sudo mount -o remount,ro /boot
```
來避免 boot 在複製的過程中被其他程序更改。再來要做的是把 `/boot` 的內容移到 `/`（root） 裏面來加密 `/boot`。由於安裝時 boot 在自己的磁盤表（ext4）而不是 `nvme0n1p6` 裏，把 `/boot`複製進 `/` 可以利用 root 來加密 boot。切記初始安裝的時候 `/boot` 必須爲 ext4 而不是 crypto，並且切勿在這一步後移除 `/boot`。然後執行：
```sh
$ sudo mount -o remount,ro /boot
$ sudo cp -axT /boot /boot.tmp
$ sudo umount /boot/efi && sudo umount /boot
$ sudo rmdir /boot
$ sudo mv -T /boot.tmp /boot
$ sudo mount /boot/efi
```
如果以上指令沒有問題，更改 `/etc/fstab` 將 `/boot` 磁盤表移除掉（由於系統在 root 裏面已經有複製過的 boot，所以不用擔心 GRUB 找不到 Debian 的 bootloader）：
```
#UUID=... /boot           ext4    defaults        0       2
```
這裏只是把 boot 的磁盤表 comment 掉，而不是移除磁盤表本身。然後在 `/etc/default/grub` 內加入
```
GRUB_ENABLE_CRYPTODISK=y
```
並執行
```sh
$ sudo update-grub
$ sudo grub-install /dev/nvme0n1
$ sudo grep 'cryptodisk\|luks' /boot/grub/grub.cfg
```
其中 `/dev/nvme0n1` 是 root（`/dev/nvme0n1p6`）和 boot（`/dev/nvme0n1p7`）的主磁盤。

最後一行應該包含 `insmod cryptodisk` 和 `insmod luks` 來代表 `/boot` 已經被加密。如果 `update-grub` 有輸出錯誤，確認安裝時的 USB 已經移除，不然系統會認定該裝置爲另一個作業系統而嘗試更新它的 GRUB。只要最後一行沒有執行錯誤就沒有問題。沒有問題的話就重新啓動。

雖說 cryptodisk 如果沒有顯現可以手動更改 `/etc/default/grub`，但個人不建議此做法。如果 cryptodisk 沒有輸出，嘗試再跑一次 `update-grub` 和 `grub-install`，還有問題的話建議檢查有沒有執行錯誤的指令。

> 重新啓動後的狀態應該是 GRUB, `/`，和 `/home` 各需要一次密碼，總計三次輸入

### 使用者密鑰
目前要輸入三次密碼，但理想狀態是在 GRUB 輸入一次密碼，然後讓 Debian 自動載入 root 和 home 的加密磁盤（直接跳進使用者的登入提示）。由於目前系統只有一個 Key slot，使用者可以生成自己的密鑰。照樣先從 root 開始：
```
$ sudo dd bs=512 count=4 if=/dev/random of=/keyfile iflag=fullblock
$ sudo chmod 600 /keyfile
$ sudo cryptsetup luksAddKey /dev/nvme0n1p6 /keyfile
$ sudo cryptsetup luksDump /dev/nvme0n1p6
```
重申一下，`nvme0n1p6` 是 root 的磁盤表，不要把 keyfile 加入 boot 或是 home（home 後面會再加入另一個密鑰，但是要先生成 `/` 的密鑰，不然 home 的密鑰無法正確載入）。現在 `luksDump` 應顯示 `Key Slot 0` 和 `Key Slot 1` 處於 ENABLED 狀態，而其他六個顯示 DISABLED。將來如果密鑰不能使用，可以嘗試生成一個新的密鑰（會自己進到 `Key Slot 2`）並把無法使用的 `Key Slot 1` 刪除。

再來更改 `/etc/crypttab`（由於我們生成的密鑰在 slot 1，所以輸入 key-slot=1）前面的 `nvme0n1p6_crypt` 會因不同裝置而有不同的名稱，但通常是 root 磁盤表的名字加上 `_crypt`：
```
nvme0n1p6_crypt UUID=<a_long_string_of_characters> /keyfile luks,discard,key-slot=1
```
並更改 `/etc/cryptsetup-initramfs/conf-hook`：
```
KEYFILE_PATTERN="/keyfile"
```
並在 `/etc/initramfs-tools/initramfs.conf` 加入：
```
UMASK=0077
```
最後更新 initramfs：
```
$ sudo update-initramfs -u -k all
```
到這一步時，root 應該已經有自己的密鑰。這時可以使用以下的指令檢查：
```
$ stat -L -c "%A  %n" /initrd.img
-rw-------  /initrd.img
$ sudo lsinitramfs /initrd.img | grep "^cryptroot/keyfiles"
cryptroot/keyfiles
cryptroot/keyfiles/nvme0n1p6_crypt.key
```
和上一步一樣，如果 keyfiles 沒有正確生成，建議回去檢查 /keyfiles 有沒有在 conf-hook 裏面更改，並且嘗試重新生成 grub 以及 initramfs。

如果到這邊都還沒有問題，接下來可以生成 home 的密鑰。這一步不用重啓電腦。執行：
```
$ sudo dd bs=512 count=4 iflag=fullblock if=/dev/random of=/crypthome.key
$ sudo chmod 400 /crypthome.key
```
如上，把這個密鑰加入 home（不是 root，因爲它已經有自己的密鑰）：
```
$ sudo cryptsetup luksAddKey /dev/nvme0n1p7 /crypthome.key
```
確認密鑰有被成功加入：
```
$ sudo cryptsetup luksDump /dev/nvme0n1p7
...
Keyslots:
  0: luks2
...
  1: luks2
...
```
這裏不用把 home 格式化到 luks1，因爲沒有要把 `/boot` 載入到 `/home` 裏面。這個手冊使用不同的 root 以及 home 磁盤表是因爲如果其中一個出現問題（例如 apt 代碼安裝或是更新 KDE 出包）不用把整個作業系統重新安裝。

跟 root 一樣，現在 home 的 `Key Slot 0` 和 `Key Slot 1` 應該都有一個密鑰。`Key Slot 0` 是最初安裝 Debian 時設定的密碼，而 Key Slot 1 是讓 Debian 在使用者輸入 GRUB 密碼後自動解鎖的密鑰。最後更改 `/etc/crypttab`：
```
nvme0n1p7_crypt UUID=<a_long_string_of_characters> /crypthome.key luks,discard,key-slot=1
```
然後重啓電腦。如果安裝順利，只需要在 GRUB 頁面輸入一個密碼，Debian 就會自動解鎖 root 和 home 並直接進入 tty 要求使用者登入賬號。

## KDE 安裝
目前的系統還在 tty，沒有一個桌面系統。個人喜好是使用 KDE；之前使用過 Gnome，但是感覺有點太像 Mac（偏好類似 Windows 的 taskbar 並移除屏幕上方的 navbar），並且 Gnome 的半透明以及其他視覺效果佔用了不少記憶體。雖然 KDE 限制比較多（例如多語言輸入非常偏好 `fcitx` 而不是 `dbus`，系統電池控制只限於 `power-profiles-daemon` 而 `tlp` 沒有相對應的控制）但是也維持了外觀及功能的一致性。

值得一提的點是 Gnome 是基於 gdm 而 KDE 是基於 sddm 的 display manager。這邊不多做比較，但重點是 Gnome 的外觀及應用程序不一定能在 KDE 運行，反之亦然。同時個人感覺 KDE 的筆電支持，例如控制板以及觸控屏幕，比 Gnome 以及大多數的桌面環境好。

### 網路控制
KDE 使用的是 NetworkManager。安裝：
```sh
$ sudo apt install network-manager network-manager-openvpn network-manager-config-connectivity-debian
```
這邊先不要重新啓動，因爲 `network-manager` 和 Debian 本身的設定有衝突，後面再做解釋。目前網路應該是能用的（可以嘗試 `ping`）。接下來安裝 KDE：
```sh
$ sudo apt install kde-plasma-desktop
```
這個下載約 3GB。每一行的狀態應該都是 `GET`；如果開始出現 `IGN`，建議終止安裝，跑 `sudo apt autoremove` 來移除已經安裝的插件，並確認網路是否連接。如果無法連接，建議跑 `sudo systemctl disable NetworkManager.service` 並且重啓電腦。

安裝 KDE 時建議不要移除 `konqueror` 或 `zutty`，因爲 `konqueror` 是 `kde-baseapps` 的上源，而 `kde-baseapps` 是 `kde-plasma-desktop` 的上源。其中移除 konqueror 就算沒有報錯也不代表 apt 的 package tree 沒有崩壞，沒必要爲了省下一點空間冒險。

由於安裝了 `network-manager`（KDE 的網路工具依賴它），必須把 Debian 本身的設定移除。這時開啓 `/etc/network/interfaces` 並把裏面每一行標註或是移除。把這一步留到現在是因爲，如果在安裝完 `network-manager` 立刻把 interfaces 的設定移除，Debian 會無法連接到網路。這是由於 `network-manager` 不會讀取 `/etc/network/interfaces` 裏的設定，所以無法連接到之前的網路（`network-manager` 的 service 在安裝完後會立刻執行，所以會跟 Debian 起衝突）。

到這步䠫就可以重啓電腦使用 KDE 並快進到下一個章節了。以下加裝只是推薦：

### 其他安裝
列印韌體：
```sh
$ sudo apt install cups print-manager
$ sudo usermod -aG lpadmin (username)
$ sudo usermod -aG lp (username)
```
這邊注意韌體不是印表機特定的驅動程式，而是一個通用的驅動。這要看廠商有沒有提供另外的 cups 驅動：如果有些設定在 Windows 或是 Mac 上能使用但是 Debian 上沒有，那就是代表那些設定只有安裝客製的 cups 驅動才能使用，而非通用的 `cups` 包裹。

防火牆：
```sh
$ sudo apt install ufw plasma-firewall
$ sudo ufw enable
```

Splash screen（載入作業系統時的特效，而不是 tty 的黑屏）：編輯 `/etc/default/grub`，找到 `GRUB_CMDLINE_LINUX_DEFAULT` 並在後面加上 `splash`。執行 `sudo update-grub` 並且重新啓動後就可以看到載入特效了。這個步䠫不會要求另外的安裝。

再多下載一個火狐：
```sh
$ sudo apt install firefox-esr
```
