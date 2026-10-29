# postgresql-1C-arch

PostgreSQL patched for 1C:Enterprise (official RPM builds from 1C), converted with `rpmtoarch` into
Arch packages (`*.pkg.tar.zst`) for Arch-based distributions (Arch, Artix, …), installed with `pacman`/`yay`.

Each release lives in its own folder:

```
postgresql-1C-pkgs/<pgver>-<1Crel>_x86_64_server/*.pkg.tar.zst
```

| Folder | Source RPM build |
|---|---|
| `18.4-1_x86_64_server` | PostgreSQL 18.4-1.1C, CentOS Stream 9 (gcc 11.5, ICU 67, OpenSSL 3) |
| `18.1-2_x86_64_server` | PostgreSQL 18.1-2.1C |

> Package names contain the version (`postgresql18-server-18_4-1_1C`), so pacman treats every
> version as a **different package** with the same files. Upgrade = remove old packages, install new ones (see §6).
> Packages carry no pacman dependencies — missing system libraries are found with `ldd` (§3).

All commands below are run from the directory where the repo is cloned. Set the variables once per shell session:

```bash
REPO=postgresql-1C-arch
PKGDIR=$(ls -d $REPO/postgresql-1C-pkgs/*_x86_64_server | sort -V | tail -1)   # latest; or set explicitly
MAJ=$(basename $PKGDIR | cut -d. -f1)                                           # 18
PGDATA=/home/admin/Data/DATABASE/PostgreDB                                      # your data dir
PGPORT=49172
echo "$PKGDIR  major=$MAJ  PGDATA=$PGDATA"
```

## 0 - clone repo

```bash
git clone https://github.com/b0ris8/postgresql-1C-arch.git
```

## 1 - install packages

```bash
sudo pacman -U $PKGDIR/*.pkg.tar.zst
```

`llvmjit` needs the exact LLVM version it was built with (18.4 → `libLLVM.so.20.1`). If your LLVM is newer,
skip that package or install it anyway — without its library PostgreSQL simply runs without JIT
(for 1C `jit = off` is recommended anyway).

## 2 - (only PostgreSQL ≤ 15) legacy libs

Old 1C builds (14.x/15.x, el8) needed extra libs that are not in Arch/AUR in the right version:

```bash
yay -S libicu50 libldap24 enchant1.6
sudo cp    $REPO/libs/openssl10/etc/pki/openssl10.cnf /etc/pki/openssl10.cnf
sudo cp -r $REPO/libs/openssl10/usr/share/  /usr/share/
sudo cp -r $REPO/libs/openssl10/usr/lib64/* /usr/lib/
```

PostgreSQL 18 builds use OpenSSL 3 and do **not** need this.

## 3 - check missing system libraries

```bash
find /usr/pgsql-$MAJ -type f \( -name postgres -o -name '*.so*' \) -exec ldd {} + 2>/dev/null \
  | awk '/not found/{print $1}' | sort -u
```

Empty output = all good. For every missing soname find the package and install it:

```bash
sudo pacman -Fy
pacman -F libexample.so.1        # official repos
yay -Ss <name>                   # if not in official repos — AUR
```

Libraries required by the 18.4 build (for reference): server — `libicui18n.so.67 libicuuc.so.67 liburing.so.2 libnuma.so.1
liblz4.so.1 libzstd.so.1 libxml2.so.2 libldap.so.2 libpam.so.0 libgssapi_krb5.so.2 libssl.so.3 libsystemd.so.0`;
contrib additionally `libxslt.so.1 libuuid.so.1 libselinux.so.1` (+ `libperl.so.5.32`, `libpython3.9.so.1.0`
only for the perl/python transform modules, not needed by 1C).

## 4 - create PGDATA and initialize cluster

Data dir on a separate drive (example: `/home/admin/Data` is the mount point):

```bash
sudo mkdir -p $PGDATA
sudo setfacl -m u:postgres:x /home/admin                 # postgres must traverse the home dir if PGDATA is inside it
sudo chown postgres:postgres $PGDATA && sudo chmod 700 $PGDATA

sudo -u postgres /usr/pgsql-$MAJ/bin/initdb -D $PGDATA --locale=ru_RU.UTF-8 -E UTF8
sudo cp $REPO/postgresql.conf $REPO/pg_hba.conf $PGDATA/
sudo chown postgres:postgres $PGDATA/postgresql.conf $PGDATA/pg_hba.conf
```

Before the first start adjust `postgresql.conf` to the machine: `shared_buffers` (≈25% RAM), `port`, `jit`.
`pg_hba.conf` in the repo is `trust` on localhost only — tighten it if 1C connects over the network.

## 5 - service

**dinit** (Artix):

```bash
sed -e "s|/usr/pgsql-[0-9]*|/usr/pgsql-$MAJ|" -e "s|-D [^ ]*|-D $PGDATA|" \
    $REPO/postgresql-1C | sudo tee /etc/dinit.d/postgresql-1C
sudo dinitctl enable postgresql-1C
```

**systemd**: a unit with `User=postgres` and
`ExecStart=/usr/pgsql-<MAJ>/bin/postgres -D <PGDATA>`, then `sudo systemctl enable --now postgresql-1C`.

Check:

```bash
psql -p $PGPORT -U postgres -Atc 'select version()'
```

## 6 - upgrade to a newer minor version (same major, e.g. 18.1 → 18.4)

Data files are compatible, no dump/pg_upgrade needed. Rollback = the same steps with the old folder.

```bash
# 1C server must be stopped first
sudo dinitctl stop --force postgresql-1C                      # --force: other services depend on it (exporter etc.)

pacman -Qq | grep "^postgresql$MAJ"                           # what is installed now
sudo pacman -Rdd $(pacman -Qq | grep "^postgresql$MAJ")       # remove ALL old ones, incl. packages absent in new build
sudo pacman -U $PKGDIR/*.pkg.tar.zst

sudo dinitctl start postgresql-1C
sleep 3; psql -p $PGPORT -U postgres -Atc 'select version()'
```

Then start the services that depend on PostgreSQL (`dinitctl list`) and the 1C server.

Collation check (needed after **glibc** updates, independent of PostgreSQL version):

```bash
psql -p $PGPORT -U postgres -Atc "select datname, datcollversion, pg_database_collation_actual_version(oid)
  from pg_database where datcollversion is distinct from pg_database_collation_actual_version(oid)"
# for each listed db (template0 with empty version is fine):
#   psql -p $PGPORT -U postgres -d <db> -c "REINDEX DATABASE <db>" -c "ALTER DATABASE <db> REFRESH COLLATION VERSION"
```

Major upgrade (e.g. 17 → 18) is different: new `/usr/pgsql-<new>` is installed alongside, then `pg_upgrade`
or dump/restore of the 1C bases via `.dt`.

## 7 - building a new version (maintainer)

1. Download from releases.1c.ru the **CentOS Stream / el9** archive (`postgresql_<ver>_centosstream_9_x86_64_package.tar.bz2`) —
   same toolchain/ICU as previous builds; other RPM distros (RED OS, Alt, Mos.OS) have different sonames.
2. Convert and build (packages are only built, not installed):

```bash
NEW=18.5-1                                     # <pgver>-<1Crel>
mkdir -p ~/build/pg$NEW && cd ~/build/pg$NEW
tar -xjf /path/to/postgresql_*_centosstream_9_x86_64_package.tar.bz2
for rpm in *.rpm; do d="${rpm%.x86_64.rpm}"; mkdir -p "$d" && mv "$rpm" "$d"/; ( cd "$d" && rpmtoarch "$rpm" ); done
for d in */; do ( cd "$(find "$d" -name PKGBUILD -printf '%h\n' | head -1)" && makepkg -f ) || echo "FAIL: $d"; done
find . -name '*.pkg.tar.zst' -printf '%s\t%f\n'
```

3. Publish:

```bash
cd ~/WorkDev/repositories/GitHub/postgresql-1C-arch
mkdir -p postgresql-1C-pkgs/${NEW}_x86_64_server
find ~/build/pg$NEW -name '*.pkg.tar.zst' -exec cp {} postgresql-1C-pkgs/${NEW}_x86_64_server/ \;
git add postgresql-1C-pkgs/${NEW}_x86_64_server && git commit -m "PostgreSQL $NEW 1C" && git push

TAG=postgresql-$(echo $NEW | tr - .)           # postgresql-18.5.1
TOKEN=$(printf 'protocol=https\nhost=github.com\n\n' | git credential fill | sed -n 's/^password=//p')
curl -sS -X POST -H "Authorization: Bearer $TOKEN" -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/b0ris8/postgresql-1C-arch/releases \
  -d "{\"tag_name\":\"$TAG\",\"target_commitish\":\"main\",\"name\":\"postgresql_$(echo $NEW | tr - .)-1C\",\"make_latest\":\"true\",\"body\":\"update to $(echo $NEW | tr - .) version\"}" \
  | grep -E '"(html_url|message)"'
unset TOKEN
```
