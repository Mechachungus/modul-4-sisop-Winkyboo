# FUSecure - FUSE-based Secure Filesystem

## Source Code Reference

```c
#define FUSE_USE_VERSION 31

#include <fuse3/fuse.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>
#include <dirent.h>
#include <sys/stat.h>
#include <pwd.h>

static const char *source_dir = "/home/shared_files";

char *get_username(uid_t uid) {
    struct passwd *pw = getpwuid(uid);
    if (pw == NULL) return NULL;
    return pw->pw_name;
}

static void fullpath(char fpath[PATH_MAX], const char *path) {
    snprintf(fpath, PATH_MAX, "%s%s", source_dir, path);
}

static int fs_getattr(const char *path, struct stat *stbuf, struct fuse_file_info *fi) {
    (void) fi;
    char fpath[PATH_MAX];
    fullpath(fpath, path);
    int res = lstat(fpath, stbuf);
    if(res == -1) return -errno;
    return 0;
}

static int fs_readdir(const char *path, void *buf, fuse_fill_dir_t filler, off_t offset, struct fuse_file_info *fi, enum fuse_readdir_flags flags) {
    (void) offset;
    (void) fi;
    (void) flags;
    char fpath[PATH_MAX];
    fullpath(fpath, path);
    DIR *dp;
    struct dirent *de;
    dp = opendir(fpath);
    if (dp == NULL) return -errno;
    while ((de = readdir(dp)) != NULL) {
        struct stat st;
        memset(&st, 0, sizeof(st));
        st.st_ino = de->d_ino;
        st.st_mode = de->d_type << 12;
        filler(buf, de->d_name, &st, 0, 0);
    }
    closedir(dp);
    return 0;
}

static int fs_open(const char *path, struct fuse_file_info *fi){
    char fpath[PATH_MAX];
    fullpath(fpath, path);

    if ((fi->flags & O_ACCMODE) != O_RDONLY)
        return -EACCES;

    uid_t uid = fuse_get_context()->uid;
    char *user = get_username(uid);

    if (strncmp(path, "/private_yuadi/", 15) == 0 && strcmp(user, "yuadi") != 0)
        return -EACCES;

    if (strncmp(path, "/private_irwandi/", 17) == 0 && strcmp(user, "irwandi") != 0)
        return -EACCES;

    int fd = open(fpath, O_RDONLY);
    if (fd == -1)
        return -errno;

    close(fd);
    return 0;
}

static int fs_read(const char *path, char *buf, size_t size, off_t offset,
                   struct fuse_file_info *fi) {
    (void) fi;
    char fpath[PATH_MAX];
    fullpath(fpath, path);

    uid_t uid = fuse_get_context()->uid;
    char *user = get_username(uid);

    if (strncmp(path, "/private_yuadi/", 15) == 0 && strcmp(user, "yuadi") != 0) return -EACCES;

    if (strncmp(path, "/private_irwandi/", 17) == 0 && strcmp(user, "irwandi") != 0) return -EACCES;

    int fd = open(fpath, O_RDONLY);
    if (fd == -1) return -errno;
    int res = pread(fd, buf, size, offset);
    if (res == -1) res = -errno;

    close(fd);
    return res;
}

static int fs_mkdir(const char *path, mode_t mode){ return -EROFS; }
static int fs_rmdir(const char *path){ return -EROFS; }
static int fs_create(const char *path, mode_t mode, struct fuse_file_info *fi){ return -EROFS; }
static int fs_write(const char *path, const char *buf, size_t size, off_t offset, struct fuse_file_info *fi) { return -EROFS; }
static int fs_unlink(const char *path){ return -EROFS; }
static int fs_rename(const char *from, const char *to, unsigned int flags){ return -EROFS; }

static struct fuse_operations fs_oper = {
    .getattr = fs_getattr,
    .readdir = fs_readdir,
    .open    = fs_open,
    .read    = fs_read,
    .mkdir   = fs_mkdir,
    .rmdir   = fs_rmdir,
    .create  = fs_create,
    .write   = fs_write,
    .unlink  = fs_unlink,
    .rename  = fs_rename,
};

int main(int argc, char *argv[]) {
    umask(0);
    return fuse_main(argc, argv, &fs_oper, NULL);
}
```
## Struktur Sistem

### Struktur Direktori Sumber

Direktori dasar berisi:

```
/home/shared_files/
├── public/
├── private_yuadi/
└── private_irwandi/
```

* `public/`: dapat diakses oleh semua user
* `private_yuadi/`: hanya dapat diakses oleh user `yuadi`
* `private_irwandi/`: hanya dapat diakses oleh user `irwandi`

### User yang Terlibat

* `yuadi` — user yang ingin melindungi jawaban praktikumnya
* `irwandi` — user yang dapat mengakses materi publik dan folder privat miliknya sendiri

## Aturan Akses yang Diterapkan

| Folder             | Dapat Diakses oleh | Hak Akses |
| ------------------ | ------------------ | --------- |
| public/            | Semua user         | Read-only |
| private\_yuadi/    | Hanya `yuadi`      | Read-only |
| private\_irwandi/  | Hanya `irwandi`    | Read-only |
| Seluruh filesystem | Semua orang        | Read-only |

Tidak ada user, bahkan root, yang dapat membuat, menulis, mengganti nama, atau menghapus file atau folder di dalam mount point.

## Cara Kerja

FUSecure menimpa beberapa operasi FUSE yang dipilih:

* `getattr`, `readdir`, `open`, `read`: memperbolehkan akses baca
* `write`, `mkdir`, `rmdir`, `create`, `unlink`, `rename`: semua mengembalikan `EROFS` (read-only error)

ID user diperiksa menggunakan:

```c
uid_t uid = fuse_get_context()->uid;
```

Akses ke folder privat dibatasi dengan mencocokkan UID user terhadap awalan nama direktori.

## Instruksi Setup

### 1. Instalasi Dependensi

```bash
sudo apt install libfuse3-dev build-essential
```

### 2. Membuat User dan Direktori

```bash
sudo useradd -m yuadi
sudo useradd -m irwandi
sudo passwd yuadi
sudo passwd irwandi

sudo mkdir -p /home/shared_files/public /home/shared_files/private_yuadi /home/shared_files/private_irwandi
sudo chown -R root:root /home/shared_files
```

### 3. Kompilasi Program

```bash
gcc fusecure.c -o fusecure `pkg-config fuse3 --cflags --libs`
```

### 4. Mount Filesystem

Jalankan sebagai user target:

```bash
su - yuadi
./fusecure /mnt/secure_fs -f -o allow_other
```

Pastikan `/mnt/secure_fs` sudah ada dan dimiliki oleh user yang menjalankan `fusecure`.

## Contoh Eksekusi

### Akses oleh `yuadi`

```bash
cat /mnt/secure_fs/public/materi.txt            # Berhasil
cat /mnt/secure_fs/private_yuadi/answer.c       # Berhasil
cat /mnt/secure_fs/private_irwandi/tugas.c      # Ditolak (Permission Denied)
rm /mnt/secure_fs/public/materi.txt             # Ditolak (Permission Denied)
```
Dokumentasi Akses Yuadi:
![Screenshot 2025-06-22 214136](https://github.com/user-attachments/assets/1bc49d62-4aff-4fa0-b30b-705576502866)

### Akses oleh `irwandi`

```bash
cat /mnt/secure_fs/public/materi.txt            # Berhasil
cat /mnt/secure_fs/private_irwandi/tugas.c      # Berhasil
cat /mnt/secure_fs/private_yuadi/answer.c       # Ditolak (Permission Denied)
mkdir /mnt/secure_fs/testfolder                 # Ditolak (Permission Denied)
```
Dokumentasi Akses Irwandi:
![Screenshot 2025-06-22 214007](https://github.com/user-attachments/assets/1a7efbfa-4c19-4611-84d3-d205510867f5)

## Daftar File

* `fusecure.c` : Source code utama dalam bahasa C
* `README.md` : Dokumentasi ini

## Catatan

* Selalu jalankan program FUSE sebagai user yang sesuai (`yuadi` atau `irwandi`)
* Mount point harus dimiliki oleh user yang menjalankan `fusecure`
* Jangan hanya menggunakan `su` untuk debugging — FUSE memeriksa UID sebenarnya, bukan shell user

## Lisensi

Ini adalah proyek mahasiswa untuk mata kuliah Sistem Operasi.

## Penulis

Mel Lo — dengan karakter `yuadi` dan `irwandi` sebagai bagian dari skenario.
