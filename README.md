# postgresql-14_5-3-1C-arch
#PostgreSQL packages patched for 1C Enterprise on Arch-based distributions 

#(these are rpm packages from oficial site 1C Enterprise, converted via utility 'rpmtoarch' for installing on Arch-based distributions with **pacman** or **yay** command)

>#**0 - additional libs needed to start current version postgresql**
```bash
yay -S libicu50 libldap24 enchant1.6
```
>#**1 - clone repo**
```bash
git clone https://github.com/boris3812/postgresql-14_5-3-1C-arch.git
```
>#**2 - install postgresql server**
```bash
yay -U postgresql-14_5-3-1C-arch/postgresql-14.5-3.1C_x86_64/*.pkg.tar.zst
```
>#**3 - install postgresql addons**
```bash
yay -U postgresql-14_5-3-1C-arch/postgresql-14.5-3.1C_x86_64_addon/*.pkg.tar.zst
```
>#**4 - create database folder (PGDATA) in separate drive** for example  '/home/admin/Data' is mount point for drive
```bash
mkdir -p /home/admin/Data/DATABASE/PostgreDB/
```
>#allow postgres user modify files on new drive (if mountpoint of new drive is in home directory - we must allow all our home directory)
```bash
sudo setfacl -m u:postgres:rwx /home/admin
```
>#make owned PGDATA folder for postgres user (this folder we also indicate in ExecStart command in our postgre service unit)
```bash
sudo chown postgres:postgres /home/admin/Data/DATABASE/PostgreDB/

sudo chmod 750 /home/admin/Data/DATABASE/PostgreDB/
```
>#**5 - enable postgresql service**
```bash
sudo cp postgresql-14-1C /etc/dinit.d/postgresql-14-1C

sudo dinitctl enable postgresql-14-1C
```
>#for systemd unit will be:
```bash
#sudo systemctl enable postgresql-14.service
```
