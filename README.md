# Arch Linux Manual Installation – Axioo Pongo 725

Dokumen ini **bukan sekadar daftar perintah**, tetapi panduan untuk **memahami struktur Linux yang benar**, sehingga setiap keputusan instalasi **punya alasan teknis** dan berdampak pada **stabilitas hardware Clevo (Axioo Pongo 725)**.

---

## 1. Cara Membaca Dokumen Ini

Setiap tahap menjawab 3 pertanyaan:

1. **Apa yang dilakukan**
2. **Kenapa perlu**
3. **Apa dampaknya ke stabilitas hardware & sistem**

Jika suatu perintah terasa panjang atau kompleks, itu disengaja—karena di situlah *kontrol* berada.

---

## 2. Mental Model Linux (PENTING SEBELUM INSTALL)

Sebelum instalasi, pahami hierarki ini:

```
Firmware (BIOS/UEFI)
 └─ Bootloader (GRUB)
     └─ Kernel (linux-lts)
         └─ Initramfs (mkinitcpio)
             └─ systemd (PID 1)
                 └─ Services / Drivers / User Space
```

**Stabilitas hardware ditentukan di 3 lapisan pertama**:

* Kernel
* Initramfs
* ACPI / EC access

Desktop environment **tidak menentukan stabilitas**, hanya kenyamanan.

---

## 3. Booting Arch ISO (Live Environment)

### Apa yang Terjadi

* Anda menjalankan **Linux minimal dari RAM**
* Tidak ada konfigurasi permanen

### Kenapa Penting

* Semua keputusan disk & kernel dimulai di sini
* Tidak ada systemd aktif penuh

### Verifikasi Dasar

```bash
ls /sys/firmware/efi/efivars
```

Jika ada output → **UEFI mode aktif (WAJIB)**

---

## 4. Network (Agar Dokumentasi Bisa Dibaca Online)

```bash
ip link
```

Menunjukkan device jaringan (biasanya `wlan0` atau `eno1`)

```bash
iwctl
```

Digunakan karena:

* Arch ISO tidak pakai NetworkManager
* Lebih deterministik

Stabilitas jaringan di sini **tidak mempengaruhi sistem akhir**.

---

## 5. Disk & Filesystem (Fondasi Jangka Panjang)

### Prinsip

* Disk layout = *future constraint*
* Salah desain → reinstall

### Rekomendasi Konseptual

```
EFI System Partition  → Bootloader
Root (/)             → OS
Home (/home)         → Data & experiment
```

### Kenapa Dipisah

* OS bisa direbuild tanpa kehilangan data
* Cocok untuk eksperimen OS & security lab

### Tools Digunakan

_sebagai contoh saya menggunakan `nvme0n1` untuk SSD target, dalam kasus ini nama SSD hasil dari `lsblk` mungkin akan berbeda dengan kasus yang anda alami jadi tetap berhati-hatilah_ :
```bash
lsblk
cfdisk /dev/nvme0n1
```

`cfdisk` dipilih karena:

* Visual
* Minim kesalahan sektor

---

## 6. Filesystem Creation

### EFI

```bash
mkfs.fat -F32 /dev/nvme0n1p1
```

### Root

```bash
mkfs.ext4 /dev/nvme0n1p2
```

(ext4 dipilih karena stabil & mudah di-debug)

### Mounting

```bash
mount /dev/nvme0n1p2 /mnt
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

**Mount = menyusun realitas sistem baru**

---

## 7. Base System Installation

```bash
pacstrap /mnt base linux-lts linux-lts-headers linux-firmware intel-ucode sudo
```

### Kenapa linux-lts

* DKMS Clevo lebih stabil
* ACPI regression lebih jarang

### Kenapa intel-ucode

* Fix CPU errata
* Mempengaruhi stabilitas & power

---

## 8. Filesystem Table (fstab)

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Kenapa Penting

* Kernel **tidak tahu** disk layout tanpa ini
* Salah UUID = kernel panic

---

## 9. Chroot (Masuk ke Sistem Baru)

```bash
arch-chroot /mnt
```

Sekarang:

* Anda **hidup di OS baru**
* Semua keputusan permanen

---

## 10. Waktu, Locale, Identitas Sistem

```bash
ln -sf /usr/share/zoneinfo/Asia/Jakarta /etc/localtime
hwclock --systohc
```

```bash
nano /etc/locale.gen
# aktifkan en_US.UTF-8
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Locale mempengaruhi:

* Logging
* Tooling dev

---

## 11. Hostname & Network
_sebagai contoh nama host saya menggunakan `pongo725`_ :
```bash
echo "pongo725" > /etc/hostname
```

```bash
pacman -S networkmanager
systemctl enable NetworkManager
```

NetworkManager dipakai karena:

* Stabil di laptop
* Cocok dengan Wayland

---

## 12. Bootloader (GRUB)

```bash
pacman -S grub efibootmgr
```

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
grub-mkconfig -o /boot/grub/grub.cfg
```

Bootloader = **penghubung firmware ↔ kernel**

---

## 13. Kernel Command Line (Stabilitas Hardware)

Edit:

```bash
nano /etc/default/grub
```

Tambahkan:

_parameter yang sudah saya coba yaitu_ :
```text
acpi_osi=!
acpi_backlight=native
acpi_enforce_resources=lax
```
_dan kedepannya saya akan mencoba menambahkan parameter berikut_ :
```
intel_pstate=active
pcie_aspm=force
```

Update:

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```

Ini **inti stabilitas Clevo**.

---

## 14. User & Privilege Model

_sebagai contoh saya menggunakan nama `clay` untuk User_
```bash
useradd -m -G wheel clay
passwd clay
```

```bash
EDITOR=nano visudo
# uncomment %wheel ALL=(ALL:ALL) ALL
```

Prinsip:

* Root hanya untuk administrasi
* Daily work = user

---

## 15. Reboot & Validasi

```bash
exit
umount -R /mnt
reboot
```

Checklist setelah boot:

* Login TTY sukses
* `uname -r` = linux-lts
* Tidak ada ACPI error fatal

---

## 16. Setelah Ini (Tahap Berikutnya)

Jika tahap ini stabil, **baru masuk ke**:

* mechrevo-drivers-dkms
* tuxedo-control-center
* GPU hybrid
* Hyprland + greetd

---

## Prinsip Akhir

> Sistem Linux yang stabil **bukan hasil banyak package**,
> tetapi hasil **keputusan yang dipahami satu per satu**.

Dokumen ini adalah fondasi Anda untuk membangun sistem **yang tidak terasa asing**, karena Anda memahami setiap lapisannya.
