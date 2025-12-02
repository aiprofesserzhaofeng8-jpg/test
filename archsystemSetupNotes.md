 # Arch Linux Setup Notes

## 1. Create a Normal User (Recommended)

Running the system as **root** is dangerous and may cause system crashes. It is recommended to create a normal user for daily operations.

### Create user `w`

```bash
useradd -m -G wheel -s /bin/bash w
```

* `-m`: Create a home directory
* `-G wheel`: Add the user to the wheel group (sudo privileges)
* `-s /bin/bash`: Set Bash as the default shell

### Set password

```bash
passwd w
```

## 2. Enable sudo for the User

Run:

```bash
visudo
```

Enable the following line:

```bash
%wheel ALL=(ALL:ALL) ALL
```

This grants sudo privileges to users in the wheel group.

---

## 3. Fix Network Issues / Configure Networking

### Check network interfaces

```bash
ip link
# or
ip a
```

### Configure DHCP for a wired adapter

Create:

```bash
sudo vim /etc/systemd/network/20-wired.network
```

Add:

```ini
[Match]
Name=enp0s3

[Network]
DHCP=yes
```

Replace `enp0s3` with your actual network interface name.

### Enable network services

```bash
sudo systemctl enable systemd-networkd --now
sudo systemctl enable systemd-resolved --now
```

### Configure DNS (create symlink)

```bash
sudo ln -sf /run/systemd/resolved/stub-resolv.conf /etc/resolv.conf
```

### Restart the network interface

```bash
sudo ip link set enp0s3 up
```

### Verify

```bash
ip a
```

---

## 4. Configure pacman Repositories

Edit:

```bash
sudo vim /etc/pacman.conf
```

Enable multilib:

```ini
[multilib]
Include = /etc/pacman.d/mirrorlist
```

### Arch Linux CN repo (for users in China)

Reference: Tsinghua Tuna Mirror

Add to `pacman.conf`:

```ini
[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

### Import GPG keys

```bash
sudo pacman -Sy archlinuxcn-keyring
```

### Update system

```bash
sudo pacman -Syyu
```

---

## 5. Install an AUR Helper (paru)

```bash
sudo pacman -S paru
```

Test installation:

```bash
paru -Ss wechat
```

---

## 6. Logout and Use the New User

```bash
exit
```

Log in using the `w` user.
-
