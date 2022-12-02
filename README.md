# postgresql-14_5-3-1C
#PostgreSQL packages patched for 1C on Arch-based distributions

#1 clone repo

git clone https://github.com/boris3812/postgresql-14_5-3-1C.git

#2 install postgresql server

yay -U postgresql-14_5-3-1C/postgresql-14.5-3.1C_x86_64/*.pkg.tar.zst

#3 install postgresql addons

yay -U postgresql-14_5-3-1C/postgresql-14.5-3.1C_x86_64_addon/*.pkg.tar.zst

#4 enable postgresql service

sudo cp postgresql-14-1C /etc/dinit.d/postgresql-14-1C

sudo dinitctl enable postgresql-14-1C

#for systemd unit
#sudo systemctl enable postgresql-14.service
