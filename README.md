# Debian 設置
應該架設自己的網站，不過目前有點懶 o_O 先放在這裏將就一下（畢竟把 OS 重設了幾次覺得可能得做一個類似的手冊）

我沒有太多 Linux 的經驗，如果有問題請勿噴太慘。先說明一下不是每個 Ubuntu 的設定都可以帶入 Debian；這個設置也盡可能避免使用 apt 以外的代碼庫來維持穩定性。

這篇手冊一部分是給我自己留着，所以中間可能會省略一些信息。

## 系統初始化
摘要：使用 luks 磁盤加密，xanmod kernel,以及使用 secure boot 設置

參考來源：[DWArmstrong](https://www.dwarmstrong.org/minimal-debian/)

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
如果有輸出的話，進入 `/etc/fstab` 把那個磁盤表 comment 掉。沒有輸出的話就沒有問題。最後再執行：
```
$ sudo zramswap start
$ sudo zramctl
```

### 磁盤加密
