# Migracija XFS/LVM podataka: RAID 0 (2 diska) → RAID 10 (4 diska)

**OS:** RHEL 9.4  **Tip dokumenta:** operativna procedura (runbook) sa kontrolnim tačkama  **Verzija:** 1.0

| Polje | Vrednost |
|---|---|
| Server | `<hostname>` |
| Change / tiket | `<broj>` |
| Izvršilac / kontrolor | `<ime>` / `<ime>` |
| Prozor za rad (downtime) | `<datum, od – do>` |
| Privremeni (TMP) server | `<hostname / IP>` |
| Mount point podataka | `<npr. /data>` |

---

## 1. Opis i obim

**Trenutno stanje:** XFS fajl sistem na LVM logičkom volumenu, ispod kog su 2 fizička diska u RAID 0 (tip RAID 0 se utvrđuje u Fazi 1: LVM striping, mdadm RAID0 ili hardverski RAID kontroler).

**Ciljno stanje:** isti mount point i isti podaci na novom RAID 10 nizu od 4 nova diska (mdadm, layout `near=2`) → LVM → XFS.

**Van obima:** root/OS diskovi. Oni se ne diraju ni u jednom koraku.

**Tok podataka:**

```
 [RAID0: 2 stara diska] --rsync/ssh--> [TMP server] --rsync/ssh--> [RAID10: 4 nova diska]
          |
          +--> izvađeni, obeleženi i čuvani netaknuti = ROLLBACK kopija
```

Tokom kritičnog dela procesa podaci postoje na **dva nezavisna mesta**: na starim diskovima (van servera) i na TMP serveru.

---

## 2. Ključna pravila (obavezno pročitati pre početka)

1. **Stari diskovi se ni u jednom trenutku ne brišu, ne inicijalizuju i ne ubacuju u drugi sistem.** Na HW kontroleru se nad njima nikad ne radi *Clear foreign config* ni *Initialize*.
2. **Nova VG dobija drugačije ime od stare** (npr. `vg_data10` umesto `vg_data`). Tako u rollback scenariju nema konflikta imena VG, a XFS ima novi UUID pa ni tu nema konflikta.
3. **Destruktivne komande (`wipefs`, `parted`, `mdadm --create`) se izvršavaju isključivo nad putanjama `/dev/disk/by-id/...`**, nikad nad `/dev/sdX`, jer se imena `sdX` menjaju posle restarta i zamene diskova.
4. **Nema prelaska u sledeću fazu bez uspešne kontrolne tačke (K1–K6).** Rezultat svake kontrolne tačke se upisuje u checklist (Prilog C).
5. **Svi izlazi komandi se čuvaju u radnom direktorijumu** i kopiraju na TMP server pre fizičkog rada na serveru.
6. **Duge operacije (rsync, checksum) se pokreću isključivo u `tmux` sesiji** kako prekid SSH veze ne bi prekinuo posao.

---

## 3. Pregled faza

| Faza | Opis | Servisi | Rollback |
|---|---|---|---|
| 0 | Priprema: paketi, varijable, logovanje | rade | nije potreban |
| 1 | Inventar postojećeg stanja | rade | nije potreban |
| 2 | Priprema TMP servera i test | rade | nije potreban |
| 3 | Inicijalna kopija (1–N prolaza) | rade | nije potreban |
| 4 | **Cutover:** stop servisa, finalna sinhronizacija, verifikacija | **DOWNTIME** | trivijalan (start servisa) |
| 5 | Deaktivacija starog storage-a | **DOWNTIME** | jednostavan (Rollback R-A) |
| 6 | Fizička zamena diskova | **DOWNTIME** | hardverski (Rollback R-C) |
| 7 | Kreiranje RAID 10 | **DOWNTIME** | R-B / R-C |
| 8 | LVM + XFS + fstab | **DOWNTIME** | R-B / R-C |
| 9 | Vraćanje podataka sa TMP servera | **DOWNTIME** | R-B / R-C |
| 10 | Verifikacija | **DOWNTIME** | R-B / R-C |
| 11 | Test reboot, pokretanje servisa, monitoring | rade | R-C (uz gubitak novih upisa) |
| 12 | Završno čišćenje (posle perioda stabilnosti) | rade | — |

**Procena trajanja downtime-a** ≈ trajanje finalnog rsync prolaza + verifikacija izvora + fizički rad (30–60 min) + kreiranje niza (minuti) + **puno vraćanje podataka** + verifikacija cilja. Vraćanje podataka je obično najduži korak; realno ga izmeriti na osnovu Faze 3.

| Mreža | Realna propusnost | Približno |
|---|---|---|
| 1 GbE | ~110 MB/s | ~390 GB/h |
| 10 GbE | 300–1000 MB/s (često ograničeno diskovima) | 1–3,5 TB/h |

Veliki broj malih fajlova drastično usporava kopiranje u odnosu na ove brojke; zato je merenje u Fazi 3 obavezno.

---

## 4. Pretpostavke i tehničke odluke

- **Novi RAID 10 je softverski (mdadm)**, `--level=10 --layout=n2 --chunk=512K --metadata=1.2`, preko GPT particija tipa *Linux RAID*. Ako diskovi idu preko HW RAID kontrolera koji ostaje u RAID modu, videti **Prilog A**.
- Kapacitet RAID 10 od 4 diska ≈ **2 × kapacitet najmanjeg diska**.
- Kopiranje: `rsync -aHAXS --numeric-ids` preko SSH, što čuva vlasnike (numerički UID/GID), prava, ACL, extended atribute, SELinux kontekste, hard linkove i sparse fajlove.
- **rsync ne čuva XFS reflink deljenje blokova** — reflinkovani fajlovi posle kopiranja zauzimaju pun prostor (bitno za kapacitet).
- Zamena diskova se radi sa **ugašenim serverom** (hot-swap varijanta je navedena kao opcija).
- Mount point ostaje isti, pa aplikacije ne zahtevaju izmenu putanja.
- Na TMP serveru se rsync izvršava kao `root` (potrebno za očuvanje vlasništva). RHEL 9 podrazumevano ima `PermitRootLogin prohibit-password`, što znači da radi prijava ključem.

---

## 5. Faza 0 — Priprema (dan ili dva pre prozora)

### 5.1 Paketi (na migracionom serveru)

```bash
dnf install -y mdadm lvm2 xfsprogs rsync acl attr smartmontools tmux
dnf install -y ledmon      # opciono: ledctl za paljenje LED lampice na slotu diska
```

### 5.2 Radni direktorijum i varijable

Vrednosti prilagoditi stvarnom stanju (popunjava se posle Faze 1):

```bash
mkdir -p /root/migracija/{etc_backup,snap_src,snap_tmp,snap_new}
cat > /root/migracija/vars.sh <<'EOF'
export DATA_MNT="/data"                 # mount point podataka
export OLD_VG="vg_data"                 # postojeća VG
export OLD_LV="lv_data"                 # postojeći LV
export NEW_VG="vg_data10"               # NOVA VG - namerno drugačije ime (rollback)
export NEW_LV="lv_data"
export MD_NAME="data10"                 # novi niz -> /dev/md/data10
export TMP_HOST="tmpserver.example.local"
export TMP_PATH="/backup/migracija_$(hostname -s)"
export WORKDIR="/root/migracija"
export SSH_OPTS="ssh -T -o Compression=no -c aes128-gcm@openssh.com"
# Popunjava se u Fazi 7, posle ubacivanja novih diskova:
# NEW_DISKS=( /dev/disk/by-id/wwn-0x... /dev/disk/by-id/wwn-0x... /dev/disk/by-id/wwn-0x... /dev/disk/by-id/wwn-0x... )
EOF
source /root/migracija/vars.sh
```

### 5.3 Logovanje sesije

Svaki put kada se radi na serveru:

```bash
tmux new -s migracija            # ponovno spajanje: tmux attach -t migracija
source /root/migracija/vars.sh
script -a "${WORKDIR}/sesija_$(date +%F_%H%M).log"
```

### 5.4 Nezavisni backup

Potvrditi da postoji poslednji **uspešan i proverljiv** backup podataka iz redovnog backup sistema. TMP kopija i stari diskovi su deo migracije, ne zamena za backup.

---

## 6. Faza 1 — Inventar postojećeg stanja

### 6.1 Snimanje stanja

```bash
source /root/migracija/vars.sh
{
  echo "===== $(date) ====="; hostnamectl; uname -r
  echo "===== lsblk ====="
  lsblk -o NAME,KNAME,TYPE,SIZE,FSTYPE,UUID,MOUNTPOINTS,MODEL,SERIAL,WWN,HCTL
  echo "===== by-id / by-path ====="
  ls -l /dev/disk/by-id/ /dev/disk/by-path/
  echo "===== mdstat ====="; cat /proc/mdstat
  echo "===== mdadm scan ====="; mdadm --detail --scan
  echo "===== LVM ====="
  pvs -o pv_name,vg_name,pv_size,pv_free,pv_uuid,dev_size
  vgs -o +vg_uuid
  lvs -a -o +devices,segtype,stripes,stripe_size,lv_uuid
  lvmdevices
  echo "===== mount / XFS ====="
  findmnt "${DATA_MNT}"
  xfs_info "${DATA_MNT}"
  df -hT "${DATA_MNT}"; df -i "${DATA_MNT}"
  echo "===== fstab / cmdline ====="
  cat /etc/fstab; cat /proc/cmdline
  grubby --info=ALL | grep -E '^(kernel|args)'
  echo "===== kontroler ====="
  lspci -nn | grep -iE 'raid|sas|sata|nvme|megaraid'
  echo "===== XFS kvote ====="
  xfs_quota -x -c 'state' "${DATA_MNT}"
  echo "===== SELinux lokalna pravila ====="
  getenforce; semanage fcontext -l -C
} > "${WORKDIR}/01_inventar_pre.txt" 2>&1

# Kopije konfiguracije
cp -a /etc/fstab /etc/lvm/devices/system.devices "${WORKDIR}/etc_backup/" 2>/dev/null
cp -a /etc/mdadm.conf "${WORKDIR}/etc_backup/" 2>/dev/null
vgcfgbackup -f "${WORKDIR}/etc_backup/lvm_%s.vgcfg"
tar czf "${WORKDIR}/etc_backup/etc_lvm.tgz" /etc/lvm
```

Za svaki PV stare VG snimiti i metapodatke diskova (primer za diskove `sdb`, `sdc`, prilagoditi):

```bash
for d in sdb sdc; do
  sfdisk -d /dev/$d            > "${WORKDIR}/etc_backup/sfdisk_$d.txt" 2>&1
  mdadm --examine /dev/$d*     > "${WORKDIR}/etc_backup/mdexamine_$d.txt" 2>&1
  smartctl -i /dev/$d          > "${WORKDIR}/etc_backup/smart_$d.txt" 2>&1
done
```

### 6.2 Određivanje tipa postojećeg RAID 0

| Nalaz u inventaru | Tip | Deaktivacija u Fazi 5 |
|---|---|---|
| `lvs` → `segtype=striped`, `#Str=2`, PV-ovi su direktno 2 diska/particije | LVM striping | LVM koraci |
| `lvs` → `segtype=raid0` | LVM RAID0 | LVM koraci |
| `/proc/mdstat` sadrži `raid0`, PV je `/dev/mdX` | mdadm RAID0 + LVM | LVM + mdadm koraci |
| PV je jedan uređaj (npr. `/dev/sdb`) veličine ≈ 2 × disk, `lspci` pokazuje RAID kontroler, MODEL je npr. `PERC ...` | HW RAID0 (virtual disk) + LVM | LVM koraci + **Prilog A** |

Upisati rezultat: **Tip RAID 0 = `______________`**, PV uređaj(i) = `______________`.

### 6.3 Zavisnosti i potencijalne zamke

```bash
# Ko koristi mount point
fuser -vm "${DATA_MNT}"
systemctl list-dependencies --reverse "$(systemd-escape -p --suffix=mount "${DATA_MNT}")"

# Deljenja preko mreže
exportfs -v 2>/dev/null
grep -rsF "${DATA_MNT}" /etc/exports /etc/exports.d/ /etc/samba/smb.conf

# Reference u systemd unitima i cron-u
grep -rsF "${DATA_MNT}" /etc/systemd/system /etc/crontab /etc/cron.d /var/spool/cron

# Da li kernel cmdline zavisi od stare VG (boot bi mogao da stane bez nje)
grep -E "rd\.lvm\.(vg|lv)=${OLD_VG}" /proc/cmdline /etc/default/grub

# Enkripcija
cat /etc/crypttab 2>/dev/null
```

Obratiti pažnju na:

- **`rd.lvm.lv=vg_data/...` u kernel cmdline** – ukloniti pre Faze 6, inače boot bez starih diskova može da završi u emergency modu:
  `grubby --update-kernel=ALL --remove-args="rd.lvm.lv=${OLD_VG}/${OLD_LV}"`
- **LUKS u `/etc/crypttab`** – ako postoji, procedura se dopunjuje slojem `cryptsetup` (nije pokriveno ovim dokumentom).
- **NFS klijenti** – posle migracije fajl sistem ima novi UUID, pa klijenti dobijaju *stale file handle* i moraju da urade remount (osim ako eksport koristi fiksni `fsid=`).
- **XFS project kvote** – ako su aktivne, sačuvati `/etc/projects` i `/etc/projid`; posle vraćanja podataka ponovo pokrenuti `xfs_quota -x -c 'project -s <ime>'` i postaviti limite.
- **Lokalna SELinux pravila** (`semanage fcontext -l -C`) su na root disku i ostaju, ali lista služi za proveru posle vraćanja.

### 6.4 Kapacitet

```bash
df -B1 --output=used,iused "${DATA_MNT}"
du -sb "${DATA_MNT}"                      # prividna veličina (apparent size)
xfs_info "${DATA_MNT}" | grep -o 'reflink=[01]'
```

| Provera | Uslov | Vrednost | OK |
|---|---|---|---|
| Kapacitet novog RAID 10 (2 × najmanji disk) | ≥ zauzeto × 1,2 | | ☐ |
| Slobodno na TMP serveru | ≥ zauzeto × 1,1 (+ reflink ekspanzija) | | ☐ |
| Inode-i na TMP serveru (bitno ako je ext4) | ≥ `iused` izvora | | ☐ |
| Veličine sva 4 nova diska | iste ili zanemarljive razlike | | ☐ |

### 6.5 Mapiranje starih diskova na fizičke slotove

Povezati serijski broj iz `lsblk`/`smartctl -i` sa inventarom u iDRAC/iLO/BMC ili ga potvrditi LED lampicom (`ledctl locate=/dev/sdX`, isključivanje: `ledctl locate_off=/dev/sdX`).

| Uređaj | Slot (bay) | Model | Serijski broj | WWN | Uloga u RAID 0 |
|---|---|---|---|---|---|
| `/dev/sd_` | | | | | član 0 |
| `/dev/sd_` | | | | | član 1 |

### ✅ Kontrolna tačka K1

- ☐ Inventar sačuvan, tip RAID 0 utvrđen
- ☐ Sve zavisnosti (servisi, eksporti, cron) popisane
- ☐ Kapacitet novog niza i TMP servera zadovoljava
- ☐ Kernel cmdline ne zavisi od stare VG (ili je plan uklanjanja spreman)
- ☐ Tabela slotova popunjena i potvrđena

---

## 7. Faza 2 — Priprema TMP servera

### 7.1 Na TMP serveru

```bash
dnf install -y rsync acl attr           # ili ekvivalent za distribuciju TMP servera
TMP_PATH="/backup/migracija_<hostname_izvora>"
mkdir -p "${TMP_PATH}/data" "${TMP_PATH}/meta"
df -hT "${TMP_PATH}"; df -i "${TMP_PATH}"

# Fajl sistem mora da podržava xattr i ACL (XFS/ext4 da; NFS/CIFS mount NE koristiti)
touch "${TMP_PATH}/t" \
  && setfattr -n user.test -v 1 "${TMP_PATH}/t" && getfattr -d "${TMP_PATH}/t" \
  && setfacl -m u:12345:r "${TMP_PATH}/t" && getfacl -n "${TMP_PATH}/t" \
  && rm -f "${TMP_PATH}/t" && echo "XATTR/ACL OK"
```

### 7.2 Na migracionom serveru: SSH ključ i test

```bash
source /root/migracija/vars.sh
ssh-copy-id root@"${TMP_HOST}"
ssh root@"${TMP_HOST}" 'rsync --version | head -1; id'

# Test očuvanja atributa na jednom probnom fajlu
mkdir -p /root/rsync_test && echo test > /root/rsync_test/f
chown 12345:12345 /root/rsync_test/f; setfacl -m u:23456:rw /root/rsync_test/f
setfattr -n user.migracija -v ok /root/rsync_test/f
rsync -aHAXS --numeric-ids -e "${SSH_OPTS}" /root/rsync_test/ root@"${TMP_HOST}":"${TMP_PATH}/rsync_test/"
ssh root@"${TMP_HOST}" "stat -c '%u:%g %a' ${TMP_PATH}/rsync_test/f; getfacl -n ${TMP_PATH}/rsync_test/f; getfattr -d -m - ${TMP_PATH}/rsync_test/f"
```

Očekivano: vlasnik `12345:12345`, ACL za `23456`, `user.migracija="ok"` i `security.selinux=...`.

> **Ako je TMP server u SELinux enforcing modu i rsync javlja grešku za `security.selinux`** (`lsetxattr ... Permission denied`), dodati u sve rsync komande `--filter='-x security.selinux'`. U tom slučaju SELinux konteksti se posle vraćanja rekonstruišu sa `restorecon` (Faza 10), a izvorni `selinux.txt` snimak služi da se otkriju ručno postavljeni konteksti (`chcon`) koji nisu pokriveni pravilima.

> **Ako se na TMP server ne sme kao root**, koristiti običnog korisnika i `--rsync-path="rsync --fake-super"` u oba smera (vlasništvo i prava se čuvaju u `user.rsync.%stat` xattr-u).

### 7.3 Skripte za verifikaciju

Kreirati obe skripte na migracionom serveru i kopirati ih na TMP server.

**`snapshot.sh`** — snima broj objekata, vlasnike/prava, veličine/mtime/hardlink broj, simlinkove, ACL, xattr, SELinux i (opciono) SHA-256 svih fajlova. Svi izlazi su sortirani, pa se snimci sa različitih servera mogu direktno porediti.

```bash
cat > /root/migracija/snapshot.sh <<'EOF'
#!/bin/bash
# snapshot.sh <direktorijum> <izlazni_dir> [--checksum]
set -euo pipefail
SRC="$1"; OUT="$(realpath -m "$2")"; MODE="${3:-}"
mkdir -p "$OUT"
cd "$SRC"
export LC_ALL=C
{
  echo "files=$(find . -type f | wc -l)"
  echo "dirs=$(find . -type d | wc -l)"
  echo "symlinks=$(find . -type l | wc -l)"
  echo "other=$(find . ! -type f ! -type d ! -type l | wc -l)"
  echo "bytes=$(find . -type f -printf '%s\n' | awk '{s+=$1} END {printf "%.0f\n", s}')"
} > "$OUT/counts.txt"
find . -printf '%y %m %U:%G %p\n'           | sort > "$OUT/meta_all.txt"
find . -type f -printf '%s %Ts %n %p\n'     | sort > "$OUT/meta_files.txt"
find . -type l -printf '%p -> %l\n'         | sort > "$OUT/symlinks.txt"
{ find . -printf '%Z %p\n' 2>/dev/null | sort; } > "$OUT/selinux.txt" || true
{ find . ! -type l -print0 | sort -z | xargs -0 -r getfacl -s -p -n 2>/dev/null; } > "$OUT/acl.txt" || true
{ find . -print0 | sort -z | xargs -0 -r getfattr -h -d -m - -e hex 2>/dev/null \
    | grep -v '^security\.selinux='; } > "$OUT/xattr.txt" || true
if [[ "$MODE" == "--checksum" ]]; then
  TMPD="$(mktemp -d "$OUT/.sha.XXXXXX")"; export TMPD
  find . -type f -print0 \
    | xargs -0 -r -P"$(nproc)" -n 500 sh -c 'sha256sum -- "$@" > "$(mktemp -p "$TMPD")"' _
  cat "$TMPD"/* 2>/dev/null | sort > "$OUT/sha256.txt"
  rm -rf "$TMPD"
fi
echo "Snapshot gotov: $OUT ($(cat "$OUT/counts.txt" | tr '\n' ' '))"
EOF
```

**`compare.sh`** — poredi dva snimka.

```bash
cat > /root/migracija/compare.sh <<'EOF'
#!/bin/bash
# compare.sh <snap_A> <snap_B> [--with-selinux]
A="$1"; B="$2"; rc=0
FILES="counts.txt meta_all.txt meta_files.txt symlinks.txt acl.txt xattr.txt"
[[ -f "$A/sha256.txt" && -f "$B/sha256.txt" ]] && FILES+=" sha256.txt"
[[ "${3:-}" == "--with-selinux" ]] && FILES+=" selinux.txt"
for f in $FILES; do
  if cmp -s "$A/$f" "$B/$f"; then
    printf 'OK    %s\n' "$f"
  else
    printf 'RAZL  %s  (prvih 20 linija razlike):\n' "$f"
    diff "$A/$f" "$B/$f" | head -20
    rc=1
  fi
done
if [[ $rc -eq 0 ]]; then echo "==> SVE ISTO"; else echo "==> POSTOJE RAZLIKE - NE NASTAVLJATI"; fi
exit $rc
EOF
chmod +x /root/migracija/{snapshot,compare}.sh
scp /root/migracija/{snapshot,compare}.sh root@"${TMP_HOST}":"${TMP_PATH}/meta/"
```

**Nivoi verifikacije** (izabrati pre prozora, na osnovu obima podataka):

| Nivo | Šta obuhvata | Trošak |
|---|---|---|
| 1 – brzi | `rsync` dry-run bez razlika + `snapshot.sh` bez `--checksum` (broj, veličine, mtime, vlasnici, prava, ACL, xattr) | minuti |
| 2 – **preporučeni** | Nivo 1 + SHA-256 svih fajlova (`--checksum`) na izvoru, TMP serveru i novom nizu | puno čitanje podataka 3× |

Brzinu SHA-256 izmeriti unapred na uzorku (npr. `time sha256sum` nad nekoliko GB) i uračunati u downtime.

---

## 8. Faza 3 — Inicijalna kopija (servisi rade)

```bash
tmux attach -t migracija || tmux new -s migracija
source /root/migracija/vars.sh

time rsync -aHAXS --numeric-ids --partial --info=progress2,stats2 \
  --log-file="${WORKDIR}/rsync_pass1.log" \
  -e "${SSH_OPTS}" \
  "${DATA_MNT}/" root@"${TMP_HOST}":"${TMP_PATH}/data/"
echo "rsync exit=$?"
```

> Kosa crta na kraju izvora (`${DATA_MNT}/`) je obavezna: kopira se sadržaj direktorijuma i atributi korenskog direktorijuma, a ne direktorijum kao podfolder.

| rsync exit kod | Značenje | Akcija |
|---|---|---|
| 0 | uspešno | nastaviti |
| 24 | fajlovi nestali tokom kopiranja | normalno dok servisi rade |
| 23 | delimičan prenos (greške atributa/pristupa) | pregledati `rsync_pass1.log`, rešiti uzrok |
| ostalo | greška | analizirati pre nastavka |

Ponavljati istu komandu (prolaz 2, 3 …, sa novim imenom log fajla) dok trajanje prolaza ne postane kratko. **Trajanje poslednjeg prolaza je dobra procena finalne sinhronizacije u downtime-u.**

| Prolaz | Početak | Trajanje | Preneto (stats2) | Exit |
|---|---|---|---|---|
| 1 | | | | |
| 2 | | | | |
| 3 | | | | |

---

## 9. Faza 4 — Cutover: zaustavljanje, finalna sinhronizacija, verifikacija (POČETAK DOWNTIME-A)

### 9.1 Zaustavljanje servisa

```bash
source /root/migracija/vars.sh
systemctl stop <aplikacioni_servisi>
systemctl stop nfs-server smb 2>/dev/null      # ako se podaci dele preko mreže
fuser -vm "${DATA_MNT}"                        # mora biti prazno
```

### 9.2 Zamrzavanje fajl sistema (read-only)

```bash
mount -o remount,ro "${DATA_MNT}"
findmnt -no OPTIONS "${DATA_MNT}" | grep -qw ro && echo "RO OK"
```

Ako se javi `target is busy`, pronaći proces (`lsof +f -- "${DATA_MNT}"`) i zaustaviti ga. Read-only garantuje da se ništa ne menja između finalne kopije i verifikacije.

### 9.3 Finalna sinhronizacija

```bash
time rsync -aHAXS --numeric-ids --delete --info=progress2,stats2 \
  --log-file="${WORKDIR}/rsync_final.log" \
  -e "${SSH_OPTS}" \
  "${DATA_MNT}/" root@"${TMP_HOST}":"${TMP_PATH}/data/"
echo "rsync exit=$?"          # MORA biti 0
```

### 9.4 Verifikacija izvor ↔ TMP

**a) rsync dry-run – ne sme biti nijedne razlike:**

```bash
rsync -aHAXSn --numeric-ids --delete --itemize-changes \
  -e "${SSH_OPTS}" \
  "${DATA_MNT}/" root@"${TMP_HOST}":"${TMP_PATH}/data/" \
  | tee "${WORKDIR}/verify_src_tmp_dryrun.txt"
wc -l < "${WORKDIR}/verify_src_tmp_dryrun.txt"      # očekivano: 0
```

**b) Snimci i poređenje** (dodati `--checksum` za Nivo 2; oba snimka mogu da rade paralelno u dva tmux prozora):

```bash
# Izvor (lokalno)
"${WORKDIR}/snapshot.sh" "${DATA_MNT}" "${WORKDIR}/snap_src" --checksum

# TMP server (izvršava se na TMP serveru)
ssh root@"${TMP_HOST}" "bash ${TMP_PATH}/meta/snapshot.sh ${TMP_PATH}/data ${TMP_PATH}/meta/snap_tmp --checksum"
rsync -a -e "${SSH_OPTS}" root@"${TMP_HOST}":"${TMP_PATH}/meta/snap_tmp/" "${WORKDIR}/snap_tmp/"

# Poređenje (bez SELinux-a, jer TMP server može imati drugačiju/isključenu SELinux politiku)
"${WORKDIR}/compare.sh" "${WORKDIR}/snap_src" "${WORKDIR}/snap_tmp"
cat "${WORKDIR}/snap_src/counts.txt"
```

**c) Arhiviranje dokumentacije na TMP server:**

```bash
rsync -a -e "${SSH_OPTS}" "${WORKDIR}/" root@"${TMP_HOST}":"${TMP_PATH}/meta/workdir_izvor/"
```

### ✅ Kontrolna tačka K2 — GO / NO-GO pre rada na hardveru

- ☐ Finalni rsync exit = 0
- ☐ Dry-run: 0 linija razlike
- ☐ `compare.sh`: `SVE ISTO` (na izabranom nivou)
- ☐ Radni direktorijum i snimci kopirani na TMP server
- ☐ Kontrolor potvrdio

**NO-GO** → `mount -o remount,rw "${DATA_MNT}"`, pokrenuti servise, analizirati. Stari storage nije diran.

---

## 10. Faza 5 — Deaktivacija starog storage-a

### 10.1 Sprečavanje upisa na root disk posle restarta

Posle zamene diskova mount point će biti prazan direktorijum na root FS-u. Servisi koji bi se automatski pokrenuli pisali bi po root disku.

```bash
systemctl disable <aplikacioni_servisi>
systemctl disable nfs-server smb 2>/dev/null
```

Popis onemogućenih servisa (za ponovno uključivanje u Fazi 11): `______________________________`

### 10.2 Odmontiranje i fstab

```bash
umount "${DATA_MNT}"
findmnt "${DATA_MNT}" || echo "ODMONTIRANO"

cp -a /etc/fstab "${WORKDIR}/fstab.pre-migracija"
vi /etc/fstab            # zakomentarisati liniju za ${DATA_MNT}, dodati prefiks  #MIGRACIJA#
grep -nF "${DATA_MNT}" /etc/fstab     # linija mora početi sa #
systemctl daemon-reload

chattr +i "${DATA_MNT}"   # opciono: prazan mount point postaje nepromenljiv (mount preko njega i dalje radi)
```

### 10.3 Deaktivacija i eksport stare VG

Zabeležiti PV uređaje stare VG pre deaktivacije:

```bash
pvs -o pv_name,pv_uuid,vg_name --select vg_name="${OLD_VG}" | tee "${WORKDIR}/old_pvs.txt"
```

```bash
vgchange -an "${OLD_VG}"
lvs "${OLD_VG}"                               # atribut LV bez 'a' (neaktivan)
vgexport "${OLD_VG}"
vgs -o vg_name,vg_attr "${OLD_VG}"            # 3. karakter vg_attr = 'x' (exported)

# Uklanjanje starih PV-ova iz LVM devices fajla (RHEL 9), za svaki PV iz old_pvs.txt:
lvmdevices --deldev <PV_uredjaj>              # npr. /dev/md127 ili /dev/sdb
lvmdevices
```

### 10.4 Samo za mdadm RAID0: zaustavljanje niza

```bash
mdadm --stop /dev/md<N>
cat /proc/mdstat                              # stari niz više ne sme biti prikazan
cp -a /etc/mdadm.conf "${WORKDIR}/mdadm.conf.pre-migracija"
vi /etc/mdadm.conf                            # zakomentarisati ARRAY liniju starog niza (#MIGRACIJA#)
```

### 10.5 Kernel cmdline i initramfs

```bash
# samo ako je K1 pokazala referencu na staru VG
grubby --update-kernel=ALL --remove-args="rd.lvm.lv=${OLD_VG}/${OLD_LV}"
dracut -f
lsinitrd | grep -E 'mdadm.conf|lvm'           # informativno
```

### 10.6 Završna provera pre gašenja

```bash
lsblk -o NAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS
dmsetup ls | grep -F "${OLD_VG}" || echo "Nema aktivnih starih LV"
```

### ✅ Kontrolna tačka K3

- ☐ `${DATA_MNT}` odmontiran, fstab linija zakomentarisana
- ☐ Stara VG neaktivna i eksportovana
- ☐ (mdadm) stari niz zaustavljen, `mdadm.conf` ažuriran
- ☐ Servisi onemogućeni za automatski start
- ☐ Kernel cmdline bez reference na staru VG, `dracut -f` urađen

> Rollback u ovoj tački: **R-A** (poglavlje 17).

---

## 11. Faza 6 — Fizička zamena diskova

### 11.1 Identifikacija i gašenje

```bash
lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN,HCTL      # potvrditi serijske brojeve iz tabele 6.5
ledctl locate=/dev/sdX                            # opciono, za oba stara diska
systemctl poweroff
```

### 11.2 Vađenje starih diskova

1. Antistatička zaštita (narukvica, ESD kese).
2. Izvaditi **samo** diskove iz tabele 6.5; proveriti serijski broj na nalepnici diska.
3. Svaki disk obeležiti nalepnicom:
   `ROLLBACK – <hostname> – slot <X> – RAID0 član <N> – <datum> – NE BRISATI`
4. Diskove spakovati u ESD kese i skloniti na zaključano mesto; upisati lokaciju: `______________`

### 11.3 Ubacivanje novih diskova i start

1. Ubaciti 4 nova diska; zabeležiti slot i serijski broj svakog.
2. Ako kontroler podržava režim po disku, nove diskove postaviti u **HBA / non-RAID / JBOD** režim (za mdadm). Ako diskovi moraju ići kroz HW RAID → **Prilog A**.
3. Uključiti server; proveriti da je boot uredan (nema emergency moda) i da se podigao sa root diskova.

| Slot | Model | Serijski broj | WWN | by-id putanja |
|---|---|---|---|---|
| | | | | |
| | | | | |
| | | | | |
| | | | | |

> **Hot-swap varijanta (samo ako je podržano i odobreno):** umesto gašenja, za svaki stari disk `echo 1 > /sys/block/sdX/device/delete`, pa ga izvaditi; posle ubacivanja novih: `for h in /sys/class/scsi_host/host*/scan; do echo "- - -" > "$h"; done`.

### ✅ Kontrolna tačka K4

- ☐ Stari diskovi izvađeni, obeleženi i sklonjeni
- ☐ Server se uredno podigao sa root diskova
- ☐ Sva 4 nova diska vidljiva u sistemu (`lsblk`)

---

## 12. Faza 7 — Kreiranje RAID 10

### 12.1 Identifikacija novih diskova

```bash
tmux new -s migracija; source /root/migracija/vars.sh
script -a "${WORKDIR}/sesija_$(date +%F_%H%M).log"

# Root diskovi – NE DIRATI
lsblk -s "$(findmnt -no SOURCE /)"
pvs

lsblk -d -o NAME,SIZE,MODEL,SERIAL,WWN,ROTA,TRAN
ls -l /dev/disk/by-id/ | grep -v -- '-part'
```

Upisati by-id putanje 4 nova diska u `vars.sh` (otkomentarisati `NEW_DISKS`), pa:

```bash
source /root/migracija/vars.sh
for d in "${NEW_DISKS[@]}"; do
  echo "===== $d -> $(readlink -f "$d")"
  lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS "$d"
  blockdev --getsize64 "$d"
  wipefs -n "$d"                               # samo prikaz potpisa, ništa ne briše
  mdadm --examine "$d" 2>&1 | head -3
  smartctl -H "$d" | grep -iE 'result|status'
done | tee "${WORKDIR}/07_novi_diskovi.txt"
```

Uslovi za nastavak: nijedan disk nije deo root VG ni montiran, SMART = `PASSED`/`OK`, veličine su iste.

### 12.2 Brisanje zatečenih potpisa (samo novi diskovi)

```bash
for d in "${NEW_DISKS[@]}"; do
  mdadm --zero-superblock "$d" 2>/dev/null
  wipefs -a "$d"
done
```

### 12.3 Particionisanje

Particija se završava 100 MiB pre kraja diska, kako bi zamenski disk nominalno iste veličine (a stvarno nešto manji) mogao da se ubaci u niz.

```bash
for d in "${NEW_DISKS[@]}"; do
  parted -s -a optimal -- "$d" mklabel gpt mkpart "md_${MD_NAME}" 1MiB -100MiB set 1 raid on
done
partprobe; udevadm settle

PARTS=( "${NEW_DISKS[@]/%/-part1}" )
ls -l "${PARTS[@]}"
lsblk -o NAME,SIZE,PARTTYPENAME,PARTLABEL "${NEW_DISKS[@]}"
```

### 12.4 Kreiranje niza

Kod `layout=n2` redosled uređaja određuje ogledalne parove: **(1↔2) i (3↔4)**. Ako su diskovi raspoređeni na dva kontrolera/backplane-a, redosled treba da bude: `ctrlA-d1, ctrlB-d1, ctrlA-d2, ctrlB-d2`, tako da svaki par ima po jedan disk na svakom kontroleru.

```bash
mdadm --create "/dev/md/${MD_NAME}" \
  --level=10 --layout=n2 --raid-devices=4 \
  --chunk=512K --metadata=1.2 --bitmap=internal \
  --name="${MD_NAME}" \
  "${PARTS[@]}"
```

### 12.5 Provera niza

```bash
cat /proc/mdstat
mdadm --detail "/dev/md/${MD_NAME}" | tee "${WORKDIR}/07_md_detail.txt"
lsblk -t "/dev/md/${MD_NAME}"                   # MIN-IO = 524288, OPT-IO = 1048576
```

Očekivano u `mdadm --detail`:

| Polje | Vrednost |
|---|---|
| Raid Level | `raid10` |
| Raid Devices / Total Devices | `4` / `4` |
| Layout | `near=2` |
| Chunk Size | `512K` |
| Intent Bitmap | `Internal` |
| State | `clean, resyncing` (tokom sinhronizacije) |
| Active / Working / Failed | `4` / `4` / `0` |

### 12.6 Konfiguracija i monitoring

```bash
mdadm --detail --brief "/dev/md/${MD_NAME}" >> /etc/mdadm.conf
grep -q '^MAILADDR' /etc/mdadm.conf || echo 'MAILADDR root' >> /etc/mdadm.conf
cat /etc/mdadm.conf
dracut -f
systemctl enable --now mdmonitor
systemctl is-active mdmonitor
```

### 12.7 Inicijalna sinhronizacija

```bash
sysctl dev.raid.speed_limit_min dev.raid.speed_limit_max
sysctl -w dev.raid.speed_limit_min=100000        # privremeno (KB/s); vratiti u Fazi 12
watch -n 10 cat /proc/mdstat
```

**Odluka:** vraćanje podataka može početi tokom resync-a (podaci su i dalje na TMP serveru i na starim diskovima), ali su oba procesa sporija. Ako prozor dozvoljava, sačekati kraj resync-a.

### ✅ Kontrolna tačka K5

- ☐ `/dev/md/${MD_NAME}` je `raid10`, `near=2`, 4 aktivna uređaja, 0 failed
- ☐ ARRAY linija u `/etc/mdadm.conf`, `dracut -f` urađen, `mdmonitor` aktivan

---

## 13. Faza 8 — LVM, XFS i fstab

### 13.1 LVM

```bash
pvcreate "/dev/md/${MD_NAME}"
vgcreate "${NEW_VG}" "/dev/md/${MD_NAME}"
lvcreate -n "${NEW_LV}" -l 100%FREE "${NEW_VG}"   # ili npr. -l 90%VG za rezervu (snapshot)
lvmdevices                                         # novi md uređaj mora biti u devices fajlu
pvs; vgs; lvs -o +devices
```

### 13.2 XFS

Pre kreiranja uporediti opcije sa starim FS-om (`xfs_info` iz `01_inventar_pre.txt`: `isize`, `crc`, `reflink`, `bigtime`, `ftype`). RHEL 9.4 `mkfs.xfs` podrazumevano uključuje `crc=1, finobt=1, reflink=1, bigtime=1, inobtcount=1`.

```bash
mkfs.xfs -L "${MD_NAME}" "/dev/${NEW_VG}/${NEW_LV}"      # labela max 12 karaktera
xfs_info "/dev/${NEW_VG}/${NEW_LV}" | tee "${WORKDIR}/08_xfs_info_new.txt"
```

Provera geometrije (blokovi od 4 KiB, chunk 512 KiB, 2 data diska): **`sunit=128 blks`, `swidth=256 blks`**. Ako `mkfs.xfs` nije sam prepoznao geometriju (sunit=0), ponoviti uz `-f -d su=512k,sw=2`.

### 13.3 fstab i mount

Mount opcije preuzeti iz stare linije (`${WORKDIR}/fstab.pre-migracija`).

```bash
chattr -i "${DATA_MNT}" 2>/dev/null
mkdir -p "${DATA_MNT}"
echo "/dev/mapper/${NEW_VG}-${NEW_LV}  ${DATA_MNT}  xfs  defaults  0 0" >> /etc/fstab
findmnt --verify --verbose
systemctl daemon-reload
mount "${DATA_MNT}"
findmnt "${DATA_MNT}"; df -hT "${DATA_MNT}"
```

---

## 14. Faza 9 — Vraćanje podataka sa TMP servera

```bash
time rsync -aHAXS --numeric-ids --info=progress2,stats2 \
  --log-file="${WORKDIR}/rsync_restore.log" \
  -e "${SSH_OPTS}" \
  root@"${TMP_HOST}":"${TMP_PATH}/data/" "${DATA_MNT}/"
echo "rsync exit=$?"          # MORA biti 0
```

Ako se prenos prekine, pokrenuti istu komandu ponovo — nastavlja se od mesta prekida.
Ako je u Fazi 2 korišćen `--filter='-x security.selinux'`, isti filter se koristi i ovde.

---

## 15. Faza 10 — Verifikacija novog storage-a

### 15.1 rsync dry-run TMP → novi niz

```bash
rsync -aHAXSn --numeric-ids --delete --itemize-changes \
  -e "${SSH_OPTS}" \
  root@"${TMP_HOST}":"${TMP_PATH}/data/" "${DATA_MNT}/" \
  | tee "${WORKDIR}/verify_tmp_new_dryrun.txt"
wc -l < "${WORKDIR}/verify_tmp_new_dryrun.txt"      # očekivano: 0
```

### 15.2 Snimak novog niza i poređenje sa izvornim snimkom

```bash
"${WORKDIR}/snapshot.sh" "${DATA_MNT}" "${WORKDIR}/snap_new" --checksum

# Izvor (stari RAID0) ↔ novi RAID10, uključujući SELinux kontekste
"${WORKDIR}/compare.sh" "${WORKDIR}/snap_src" "${WORKDIR}/snap_new" --with-selinux
```

Očekivano: `SVE ISTO`.

### 15.3 SELinux (ako `selinux.txt` pokazuje razlike)

```bash
restorecon -RnvF "${DATA_MNT}" | head -50        # dry-run: šta bi se promenilo po politici
restorecon -RvF "${DATA_MNT}"                     # primena
"${WORKDIR}/snapshot.sh" "${DATA_MNT}" "${WORKDIR}/snap_new"
"${WORKDIR}/compare.sh" "${WORKDIR}/snap_src" "${WORKDIR}/snap_new" --with-selinux
```

Preostale razlike u `selinux.txt` znače ručno postavljene kontekste (`chcon`) na izvoru; vratiti ih ručno prema `snap_src/selinux.txt` ili ih trajno definisati sa `semanage fcontext`.

### 15.4 Ostale provere

```bash
df -hT "${DATA_MNT}"; df -i "${DATA_MNT}"
diff <(grep -E '^(files|dirs|symlinks|other|bytes)=' "${WORKDIR}/snap_src/counts.txt") \
     <(grep -E '^(files|dirs|symlinks|other|bytes)=' "${WORKDIR}/snap_new/counts.txt") && echo "COUNTS OK"
xfs_quota -x -c 'state' "${DATA_MNT}"             # ako su kvote korišćene – vratiti konfiguraciju
```

### ✅ Kontrolna tačka K6

- ☐ Restore rsync exit = 0
- ☐ Dry-run TMP → novi niz: 0 linija
- ☐ `compare.sh snap_src snap_new --with-selinux`: `SVE ISTO`
- ☐ Kvote / specifične postavke vraćene (ako postoje)
- ☐ Kontrolor potvrdio

---

## 16. Faza 11 — Test reboot, pokretanje servisa, monitoring

### 16.1 Test reboot (preporučeno pre pokretanja servisa)

```bash
rsync -a -e "${SSH_OPTS}" "${WORKDIR}/" root@"${TMP_HOST}":"${TMP_PATH}/meta/workdir_posle/"
systemctl reboot
```

Posle restarta:

```bash
source /root/migracija/vars.sh
cat /proc/mdstat
mdadm --detail "/dev/md/${MD_NAME}" | grep -E 'State :|Active Devices|Working Devices|Failed Devices'
lvs "${NEW_VG}"
findmnt "${DATA_MNT}"
systemctl --failed
journalctl -b -p err --no-pager | tail -50
dmesg -T | grep -iE 'md[0-9]|raid|xfs|i/o error'
```

### 16.2 Pokretanje servisa

```bash
systemctl enable --now <aplikacioni_servisi>
systemctl enable --now nfs-server smb 2>/dev/null
exportfs -ra; exportfs -v
```

- ☐ Aplikativni smoke test (vlasnik aplikacije potvrdio)
- ☐ NFS klijenti remount-ovani (očekivan *stale file handle* bez `fsid=`)
- ☐ **KRAJ DOWNTIME-A:** vreme `______`

### 16.3 Monitoring posle migracije (24–72 h)

```bash
cat /proc/mdstat                                   # resync završen
mdadm --monitor --scan --oneshot --test            # test obaveštenja (mail za MAILADDR)
for d in "${NEW_DISKS[@]}"; do smartctl -H "$d"; done
grep -E '^ENABLED' /etc/sysconfig/raid-check       # periodični raid-check
```

- ☐ Resync završen, niz `clean`
- ☐ Test mail od mdmonitor primljen
- ☐ Prvi redovni backup nove lokacije uspešan i proveren

---

## 17. Rollback procedure

### Kriterijumi za rollback

- verifikacija (K2, K6) ne prolazi, a uzrok ne može da se reši u prozoru;
- novi diskovi/niz pokazuju greške (failed disk, I/O greške u `dmesg`);
- aplikacija ne radi ispravno na novom storage-u, a uzrok je u storage-u;
- prozor za rad ističe pre završetka Faze 10.

> **Važno:** stari diskovi sadrže stanje u trenutku finalne sinhronizacije. Sve što je upisano na novi storage posle pokretanja servisa (Faza 11) **ne postoji** na starim diskovima i mora se prebaciti ručno (rsync delta) ako je potrebno.

### R-A — Rollback pre vađenja diskova (posle Faze 4 ili 5)

```bash
source /root/migracija/vars.sh
# samo za mdadm RAID0:
mdadm --assemble --scan            # ili: mdadm --assemble /dev/md<N> --uuid=<UUID iz inventara>
cp -a "${WORKDIR}/mdadm.conf.pre-migracija" /etc/mdadm.conf 2>/dev/null

# LVM
lvmdevices --adddev <PV_uredjaj>  # za svaki PV iz old_pvs.txt
vgimport "${OLD_VG}"
vgchange -ay "${OLD_VG}"

# fstab, cmdline, initramfs
chattr -i "${DATA_MNT}" 2>/dev/null
cp -a "${WORKDIR}/fstab.pre-migracija" /etc/fstab
grubby --update-kernel=ALL --args="rd.lvm.lv=${OLD_VG}/${OLD_LV}"   # samo ako je uklonjeno
dracut -f
systemctl daemon-reload
mount "${DATA_MNT}"                # ako je FS ostao ro: mount -o remount,rw
systemctl enable --now <aplikacioni_servisi>
```

### R-B — Problem sa novim nizom, TMP kopija ispravna

Hardverski rollback nije neophodan. Uraditi `umount`, `vgremove`, `mdadm --stop`, ukloniti ARRAY liniju i fstab liniju, pa ponoviti Faze 7–10. Ako prozor ističe ili je hardver novog diska neispravan → **R-C**.

### R-C — Potpuni hardverski rollback (povratak starih diskova)

**1. Deaktivacija novog storage-a**

```bash
source /root/migracija/vars.sh
systemctl stop <aplikacioni_servisi>
systemctl disable <aplikacioni_servisi>
# Ako su servisi radili na novom storage-u, prvo sačuvati izmene nastale posle cutover-a:
#   rsync -aHAXS --numeric-ids -e "${SSH_OPTS}" "${DATA_MNT}/" root@"${TMP_HOST}":"${TMP_PATH}/data_posle_cutover/"
umount "${DATA_MNT}"
vi /etc/fstab                                   # zakomentarisati liniju novog LV
vgchange -an "${NEW_VG}"
lvmdevices --deldev "/dev/md/${MD_NAME}"
mdadm --stop "/dev/md/${MD_NAME}"
vi /etc/mdadm.conf                              # zakomentarisati ARRAY liniju novog niza
dracut -f
systemctl poweroff
```

**2. Hardver**

1. Izvaditi 4 nova diska i obeležiti ih (`NOVI – RAID10 član N – slot X`).
2. Vratiti stare diskove **u iste slotove** prema tabeli 6.5 i nalepnicama.
3. Uključiti server. Kod HW RAID kontrolera: uvoz foreign konfiguracije (Prilog A.4).

**3. Aktivacija starog storage-a**

```bash
source /root/migracija/vars.sh
lsblk -o NAME,SIZE,MODEL,SERIAL,WWN              # potvrditi serijske brojeve starih diskova

# samo za mdadm RAID0:
cp -a "${WORKDIR}/mdadm.conf.pre-migracija" /etc/mdadm.conf
mdadm --assemble --scan
cat /proc/mdstat                                 # raid0 aktivan, 2 člana

lvmdevices --adddev <PV_uredjaj>                 # za svaki PV
pvscan
vgimport "${OLD_VG}"
vgchange -ay "${OLD_VG}"
xfs_repair -n "/dev/${OLD_VG}/${OLD_LV}"        # samo provera, ništa ne menja (FS nemontiran)

chattr -i "${DATA_MNT}" 2>/dev/null
cp -a "${WORKDIR}/fstab.pre-migracija" /etc/fstab
grubby --update-kernel=ALL --args="rd.lvm.lv=${OLD_VG}/${OLD_LV}"   # samo ako je uklonjeno
dracut -f
systemctl daemon-reload
mount "${DATA_MNT}"

# Brza provera (bez checksum-a) prema izvornom snimku
"${WORKDIR}/snapshot.sh" "${DATA_MNT}" "${WORKDIR}/snap_rollback"
"${WORKDIR}/compare.sh" "${WORKDIR}/snap_src" "${WORKDIR}/snap_rollback" --with-selinux

systemctl enable --now <aplikacioni_servisi>
```

> Ako `/root/migracija` nije dostupan (npr. problem sa root diskom), sve kopije konfiguracije i snimci se nalaze na TMP serveru u `${TMP_PATH}/meta/`.

---

## 18. Faza 12 — Završno čišćenje (posle perioda stabilnosti)

**Uslovi (svi moraju biti ispunjeni):** ☐ min. `___` dana stabilnog rada ☐ uspešan i proveren redovni backup nove lokacije ☐ pisana potvrda vlasnika sistema/aplikacije.

```bash
sysctl -w dev.raid.speed_limit_min=1000                     # RHEL podrazumevana vrednost
vi /etc/fstab; vi /etc/mdadm.conf                            # ukloniti #MIGRACIJA# linije
dracut -f
ssh root@"${TMP_HOST}" "rm -rf ${TMP_PATH}/data"             # TMP kopija (meta/ zadržati kao zapis)
```

- ☐ Stari diskovi bezbedno obrisani/uništeni prema internoj politici (zapisnik o uništavanju) ili vraćeni u magacin
- ☐ Dokumentacija (radni direktorijum, checklist) priložena uz change tiket

---

## Prilog A — Varijanta sa hardverskim RAID kontrolerom (Dell PERC / Broadcom MegaRAID)

Primeri koriste `perccli64`; za Broadcom kontrolere sintaksa je ista uz `storcli64`. Sintaksa se razlikuje između verzija alata, proveriti `perccli64 help`.

**A.1 Inventar**

```bash
perccli64 /c0 show
perccli64 /c0/vall show all
perccli64 /c0/eall/sall show
perccli64 /c0 show bootdrive
perccli64 /c0 show preservedcache
```

**A.2 Pre vađenja starih diskova (posle Faze 5, pre gašenja)**

- Proveriti da nema *preserved cache* za stari VD (uredno odmontiranje i gašenje prazne keš).
- **Nikada** ne raditi *Initialize*, *Clear config* ili *Delete VD* pre nego što su diskovi izvađeni.
- Posle vađenja kontroler može prijaviti VD kao *Offline/Missing*. Brisanje te definicije sa kontrolera **ne briše metapodatke na samim diskovima**; kod ponovnog ubacivanja diskovi se prikazuju kao *Foreign* i VD se vraća uvozom (A.4).

**A.3 Novi RAID 10 kao virtual disk (umesto Faze 7)**

```bash
perccli64 /c0 add vd type=raid10 drives=<enc>:<s1>-<s4> pdperarray=2 name=data10
perccli64 /c0/vall show all | grep -iE 'strip|state|size'
perccli64 /c0/v<N> show bgi                     # background initialization
perccli64 /c0 show bootdrive                    # boot VD mora ostati root VD
```

Zatim Faza 8 direktno nad VD uređajem (`pvcreate /dev/disk/by-id/<VD>`, bez mdadm koraka). Kontroler često ne prijavljuje geometriju OS-u, pa XFS geometriju zadati ručno prema veličini strip-a VD-a:
`mkfs.xfs -L data10 -d su=<strip_size>,sw=2 /dev/${NEW_VG}/${NEW_LV}`

**A.4 Rollback – uvoz foreign konfiguracije starih diskova**

Pre ubacivanja starih diskova nove diskove izvaditi, kako bi foreign konfiguracija pripadala samo starim diskovima.

```bash
perccli64 /c0/fall show                        # prikaz foreign konfiguracije
perccli64 /c0/fall import preview
perccli64 /c0/fall import
perccli64 /c0/vall show
```

---

## Prilog B — Zamena otkazalog diska u novom mdadm RAID 10

```bash
mdadm --detail "/dev/md/${MD_NAME}"                           # utvrditi faulty/removed član
mdadm "/dev/md/${MD_NAME}" --fail /dev/disk/by-id/<stari>-part1 --remove /dev/disk/by-id/<stari>-part1
# fizička zamena diska, zatim particionisanje identično Fazi 7:
parted -s -a optimal -- /dev/disk/by-id/<novi> mklabel gpt mkpart "md_${MD_NAME}" 1MiB -100MiB set 1 raid on
partprobe; udevadm settle
mdadm "/dev/md/${MD_NAME}" --add /dev/disk/by-id/<novi>-part1
watch -n 10 cat /proc/mdstat
```

Ako se promenio sastav niza, ARRAY linija u `mdadm.conf` ostaje ista (identifikuje niz po UUID-u), ali ne škodi `dracut -f`.

---

## Prilog C — Checklist za štampu

| # | Korak | Vreme | Izvršio | Kontrolisao |
|---|---|---|---|---|
| 0 | Paketi, `vars.sh`, tmux/script, potvrđen nezavisni backup | | | |
| K1 | Inventar, tip RAID 0, zavisnosti, kapacitet, tabela slotova | | | |
| 2 | TMP server: prostor, xattr/ACL test, SSH ključ, test atributa | | | |
| 3 | Inicijalni rsync prolazi, izmereno trajanje | | | |
| 4a | Servisi zaustavljeni, `fuser` prazan, FS `ro` | | | |
| 4b | Finalni rsync exit 0 | | | |
| K2 | Dry-run 0 linija, `compare.sh` SVE ISTO, workdir na TMP → **GO/NO-GO** | | | |
| 5 | Servisi onemogućeni, umount, fstab, vgexport, lvmdevices, (mdadm stop), cmdline, dracut | | | |
| K3 | Stari storage deaktiviran | | | |
| 6 | Poweroff, stari diskovi izvađeni + obeleženi + sklonjeni, novi ubačeni, boot OK | | | |
| K4 | 4 nova diska vidljiva | | | |
| 7 | Provera diskova, wipefs, particije, `mdadm --create`, mdadm.conf, mdmonitor | | | |
| K5 | RAID 10 near=2, 4/4 aktivna | | | |
| 8 | pvcreate/vgcreate/lvcreate, mkfs.xfs (sunit/swidth), fstab, mount | | | |
| 9 | Restore rsync exit 0 | | | |
| K6 | Dry-run 0, `compare.sh --with-selinux` SVE ISTO | | | |
| 11 | Test reboot OK, servisi uključeni, smoke test, NFS klijenti | | | |
| 11b | Resync gotov, mdmonitor test mail, backup nove lokacije OK | | | |
| 12 | Čišćenje posle perioda stabilnosti, sudbina starih diskova i TMP kopije | | | |
