---
title: "Manajemen Disk Menggunakan Parted"
date: 2023-02-28T00:58:29+07:00
draft: false
categories: [storage]
tags: [disk,partition]
---

## Manajemen Disk menggunakan parted
Bismillahirrahmanirrahim, pada kali ini saya akan menulis mengenai manajemen disk menggunakan utilitas atau command parted, biasanya saya menggunakan fdisk tapi kali ini saya mencoba menggunakan parted. jadi saya telah menambah disk baru sebesar **2GB** terlihat dibawah dengan nama **"vdb"**

### Step 1: list atau cek disk baru

```bash
root@scinstance1:~# lsblk 
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda     254:0    0    5G  0 disk 
├─vda1  254:1    0  4.9G  0 part /
├─vda14 254:14   0    4M  0 part 
└─vda15 254:15   0  106M  0 part /boot/efi
vdb     254:16   0    2G  0 disk
```

### Step 2: buat partisi
sebelum disk bisa digunakan dan di mounting, kita perlu partisi terlebih dahulu

```bash
parted -s /dev/vdb mklabel gpt
parted -s /dev/vdb mkpart primary 2048s  200MB
parted -s /dev/vdb print
```

Perintah di atas digunakan untuk membuat partisi baru pada disk **/dev/vdb**, menggunakan utilitas **"parted"** pada sistem Linux.
Penjelasan masing-masing perintah adalah sebagai berikut:
- **parted -s /dev/vdb mklabel gpt**: Perintah ini digunakan untuk membuat label partisi baru pada disk /dev/vdb dengan format GPT (GUID Partition Table). Label partisi ini diperlukan agar disk dapat dipartisi ke dalam beberapa partisi terpisah.

- **parted -s /dev/vdb mkpart primary 2048s 200MB**: Perintah ini digunakan untuk membuat partisi baru pada disk /dev/vdb. Argumen "primary" menunjukkan bahwa partisi yang dibuat adalah partisi utama (primary partition). Argumen "2048s" menunjukkan lokasi awal partisi dalam satuan sektor. Argumen "200MB" menunjukkan ukuran partisi yang ingin dibuat.

- **parted -s /dev/vdb print**: Perintah ini digunakan untuk mencetak informasi partisi dari disk /dev/vdb.

Berikut adalah contoh output dari perintah **parted -s /dev/vdb print** setelah perintah di atas dijalankan:

```bash
Model: Name Or Brand Device
Disk /dev/vdb: 2G
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start    End      Size     File system  Name     Flags
 1      2048s    411647s  200MB                primary
```

> note: 2048s ini kurang lebih sama dengan 1mb, jadi diatas maksudnya adalah range 1mb - 200mb kira kira akan dialokasikan untuk membuat partisi:

kita bisa lihat partisi telah terbuat kurang lebih hampir 200mb

```bash
lsblk | grep vdb
```

### Step 3: buat mount direktori dan format file sistem untuk partisi

```bash
mkdir /mnt/disk-b
mkfs.xfs /dev/vdb1
```

### Step 4: mount ke direktori

```bash
mount /dev/vdb1 /mnt/disk-b/
```

kemudian cek

```bash
root@scinstance1:~# lsblk 
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
vda 254:0 0 5G 0 disk 
├─vda1 254:1 0 4.9G 0 part /
├─vda14 254:14 0 4M 0 part 
└─vda15 254:15 0 106M 0 part /boot/efi
vdb 254:16 0 2G 0 disk 
└─vdb1 254:17 0 190M 0 part /mnt/disk-b
```

> note: untuk membuat disk agar persistent dan tidak lepas lagi setelah reboot, perlu menambah kan uuid disk atau partisi ke /etc/fstab:

### Step 4: mount persistent
untuk melihat uuid

```bash
root@scinstance1:~# blkid | grep vdb1
/dev/vdb1: UUID="f3f3c2b2-b1c9-47f5-b5ca-12e0642a62cd" TYPE="xfs"
```

kemudian tambahkan uuid tersebut ke /etc/fstab dibaris terakhir

```bash
...
    UUID="f3f3c2b2-b1c9-47f5-b5ca-12e0642a62cd"  /mnt/disk-b  xfs   defaults 0 0
...
```
> Note: "**Baris UUID="f3f3c2b2-b1c9-47f5-b5ca-12e0642a62cd" /mnt/disk-b xfs defaults 0 0**" merupakan baris konfigurasi yang akan dimasukkan ke dalam file **"/etc/fstab"**. Baris ini bertujuan untuk me-mount partisi **"/dev/vdb1"** yang telah diformat sebagai sistem **XFS** pada direktori **"/mnt/disk-b"** secara otomatis setiap kali sistem di-boot.
Penjelasan untuk setiap field pada baris tersebut adalah sebagai berikut:

- **UUID="f3f3c2b2-b1c9-47f5-b5ca-12e0642a62cd"** : UUID dari partisi "/dev/vdb1"
- **/mnt/disk-b** : direktori tujuan untuk me-mount partisi tersebut
- **xfs** : jenis sistem berkas pada partisi tersebut
- **defaults** : opsi default untuk me-mount partisi tersebut
- **0** : opsi untuk file sistem. Nilai "0" berarti partisi tidak akan di-backup oleh perintah "dump".
- **0** : opsi untuk perintah "fsck". Nilai "0" berarti partisi tidak akan diperiksa dengan perintah "fsck" pada saat booting sistem. :


kemudian reload dan mounting kembali

```bash
systemctl daemon-reload
mount -a
```

cek

```bash
df -h
```

jika kita ingin membuat partisi lagi, maka yang dialokasikan start dari 201mb sampai seterusnya. bisa cek dengan parted -l atau parted /dev/vdb print free
contoh disini kita coba alokasi kan sekitar 100mb

```bash
parted -s /dev/vdb mkpart primary 201MB 300MB
parted /dev/vdb print
```

maka akan terbuat /dev/vdb2, contoh kita akan jadikan untuk swap

```bash
mkswap /dev/vdb2 
```

untuk menghidupkan swap

```bash
swapon /dev/vdb2
swapon --show
df -h
```

### Contoh Lain
Berikut adalah contoh cara membuat partisi dengan parted dan menentukan offset dengan satuan **MB**:

```bash
parted -s /dev/sdb mkpart primary ext4 500MB 1500MB
```

Pada contoh di atas, kita membuat partisi dengan tipe **ext4** pada disk **/dev/sdb** dengan ukuran 1**000 MB (1500 MB - 500 MB)**. Offset partisi dimulai dari 500 MB menggunakan satuan MB sebagai pengukuran.
Kita juga dapat menentukan offset dengan satuan **GB**, seperti pada contoh berikut:

```bash
parted -s /dev/sdb mkpart primary ext4 1GB 2GB
```

Pada contoh di atas, kita membuat partisi dengan tipe **ext4** pada disk **/dev/sdb** dengan ukuran** 1 GB**. Offset partisi dimulai dari **1 GB**, menggunakan satuan **GB** sebagai pengukuran. Partisi ini akan berakhir pada **2 GB** pada disk **/dev/sdb**.


## Akhir
Terima Kasih yang telah membaca sampai akhir tulisan saya yang belibet ini wkwk, sekian catatan pembelajaran saya yang dimana bertujuan untuk berbagiu terkait apa yang saya pelajari semoga dapat dipahami dan bermanfaat. saran dan masukan sangat berharga sekali jika temen temen sekalian menemukan kesalahan dalam tulisan diatas. bisa langsung hubungi saja kontak yang tertera pada Beranda web ini. 