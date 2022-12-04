# postgresql-14_5-3-1C-arch
#PostgreSQL packages patched for 1C Enterprise on Arch-based distributions 

#(these are rpm packages from oficial site 1C Enterprise, converted via utility 'rpmtoarch' for installing on Arch-based distributions with **pacman** or **yay** command)

#**0 - additional libs needed to start current version postgresql**

yay -S libicu50 libldap24 enchant1.6

#**1 - clone repo**

git clone https://github.com/boris3812/postgresql-14_5-3-1C-arch.git

#**2 - install postgresql server**

yay -U postgresql-14_5-3-1C/postgresql-14.5-3.1C_x86_64/*.pkg.tar.zst

#**3 - install postgresql addons**

yay -U postgresql-14_5-3-1C/postgresql-14.5-3.1C_x86_64_addon/*.pkg.tar.zst

#**4 - create database folder in separate drive** for example  '/home/admin/Data' is mount point for drive

mkdir -p /home/admin/Data/DATABASE/PostgreDB/

#allow postgres user modify files on new drive (if mountpoint of new drive is in home directory - we must allow all our home directory)

sudo setfacl -m u:postgres:rwx /home/admin

#make owned DATA folder for postgres user (this folder we also indicate in ExecStart command in our postgre service unit)

sudo chown postgres:postgres /home/admin/Data/DATABASE/PostgreDB/

sudo chmod 750 /home/admin/Data/DATABASE/PostgreDB/

#**5 - enable postgresql service**

sudo cp postgresql-14-1C /etc/dinit.d/postgresql-14-1C

sudo dinitctl enable postgresql-14-1C

#for systemd unit will be:

#sudo systemctl enable postgresql-14.service
