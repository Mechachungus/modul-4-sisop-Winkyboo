# FUSecure - FUSE-based Secure Filesystem

## 📘 Description

FUSecure is a custom file system built using [FUSE](https://github.com/libfuse/libfuse) (Filesystem in Userspace). It is designed to simulate a secure academic environment where users can:

* **Share public course materials**
* **Keep private practicum answers safe from unauthorized access**
* **Prevent file tampering or deletion by enforcing read-only rules**

This is useful in scenarios where multiple users share a system but require access control on specific folders.

---

## 🏗️ System Structure

### 🔧 Source Directory Layout

The base directory contains:

```
/home/shared_files/
├── public/
├── private_yuadi/
└── private_irwandi/
```

* `public/`: accessible to all users
* `private_yuadi/`: accessible **only** by user `yuadi`
* `private_irwandi/`: accessible **only** by user `irwandi`

### 👥 Users Involved

* `yuadi` — user who wants to protect his practicum answers
* `irwandi` — user who can access public materials and his own private folder

---

## 🔒 Access Rules Implemented

| Folder            | Accessible by  | Permissions |
| ----------------- | -------------- | ----------- |
| public/           | All users      | Read-only   |
| private\_yuadi/   | Only `yuadi`   | Read-only   |
| private\_irwandi/ | Only `irwandi` | Read-only   |
| Entire filesystem | Everyone       | Read-only   |

No user, not even root, can create, write, rename, or delete any files or folders in the mount point.

---

## 🧠 How It Works

FUSecure overrides selected FUSE operations:

* `getattr`, `readdir`, `open`, `read`: allow read access
* `write`, `mkdir`, `rmdir`, `create`, `unlink`, `rename`: all return `EROFS` (read-only error)

User ID is checked via:

```c
uid_t uid = fuse_get_context()->uid;
```

Private folder access is restricted by comparing the user UID to the directory name prefix.

---

## ⚙️ Setup Instructions

### 1. 🔧 Install Dependencies

```bash
sudo apt install libfuse3-dev build-essential
```

### 2. 👥 Create Users & Directories

```bash
sudo useradd -m yuadi
sudo useradd -m irwandi
sudo passwd yuadi
sudo passwd irwandi

sudo mkdir -p /home/shared_files/public /home/shared_files/private_yuadi /home/shared_files/private_irwandi
sudo chown -R root:root /home/shared_files
```

### 3. 🧾 Build the Program

```bash
gcc fusecure.c -o fusecure `pkg-config fuse3 --cflags --libs`
```

### 4. 📂 Mount the Filesystem

Run as the target user:

```bash
su - yuadi
./fusecure /mnt/secure_fs -f -o allow_other
```

Ensure `/mnt/secure_fs` exists and is owned by `yuadi`.

---

## 🧪 Execution Examples

### ✅ Access from `yuadi`

```bash
cat /mnt/secure_fs/public/materi.txt            # Success
cat /mnt/secure_fs/private_yuadi/answer.c       # Success
cat /mnt/secure_fs/private_irwandi/tugas.c      # Permission Denied
rm /mnt/secure_fs/public/materi.txt             # Permission Denied
```

### ✅ Access from `irwandi`

```bash
cat /mnt/secure_fs/public/materi.txt            # Success
cat /mnt/secure_fs/private_irwandi/tugas.c      # Success
cat /mnt/secure_fs/private_yuadi/answer.c       # Permission Denied
mkdir /mnt/secure_fs/testfolder                 # Permission Denied
```

---

## 📁 File List

* `fusecure.c` : Main C source code
* `README.md` : This documentation

---

## 📌 Notes

* Always run the FUSE program as the intended user (`yuadi` or `irwandi`)
* Mount point must be owned by the user running `fusecure`
* Do not use `su` alone for debugging — FUSE checks actual UID, not shell user

---

## 📜 License

This is a student project for Operating Systems coursework.

---

## ✍️ Author

Mel Lo — with characters `yuadi` and `irwandi` as part of the scenario.
