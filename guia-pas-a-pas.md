# Guia pas a pas: Pràctica NFS amb control d'accés (Ubuntu Server + Zorin OS)

Cada pas indica la comanda, què has de capturar i què has d'anotar. Les IP, noms i sortides seran les que et surtin a tu; aquí t'explico on mirar-ho.

> **Abans de començar:** maximitza el terminal i, si cal, redueix la lletra amb `Ctrl` + `-`. Treballa per SSH des del PC amb **dues finestres una al costat de l'altra** (servidor a l'esquerra, client a la dreta): així moltes captures mostren les dues màquines alhora. Si una captura no hi cap, parteix-la en `a` i `b` (`captura-06a.png`, `captura-06b.png`).

## Què canvia respecte al que ja tenies fet

La rúbrica demana coses que la primera versió no tenia. Les canviem així:

| Abans | Ara (el que demana la rúbrica) |
|---|---|
| Adaptador pont (192.168.1.x) | Adaptador 1 **NAT** (internet) + Adaptador 2 **Només amfitrió** (192.168.56.x) |
| Usuari `usuari_smx` | Grups `devs` i `admins`, usuaris `dev01` i `admin01` |
| Una carpeta `compartit` amb `*` | Carpetes `devs` i `admins`, exportades **per IP** (RW / RO) |
| Sense prova de `root_squash` | Prova amb `root_squash` (error) i amb `no_root_squash` (funciona) |

**Valors que he triat** (si el teu enunciat en diu uns altres, canvia'ls a tot arreu):

| Dada | Servidor | Client |
|---|---|---|
| Nom de la MV i hostname | `nfs-servidor` | `nfs-client` |
| Idioma | `ca_ES.UTF-8` | `ca_ES.UTF-8` |
| IP NAT (`enp0s3`) | DHCP (10.0.2.15) | DHCP (10.0.2.15) |
| IP només amfitrió (`enp0s8`) | `192.168.56.10` | `192.168.56.20` (prova: `.30`) |

| Grup / usuari | GID / UID |
|---|---|
| `devs` | 2001 |
| `admins` | 2002 |
| `dev01` (grup `devs`) | 2101 |
| `admin01` (grup `admins`) | 2102 |

---

## Fase 1. Infraestructura i xarxa

### 1. Configurar la xarxa a VirtualBox

Amb les dues MV **apagades**:

1. A VirtualBox: **Fitxer → Eines → Gestor de xarxa → Xarxes només amfitrió**. Comprova que hi ha una xarxa amb IP `192.168.56.1/24` (si no, clica **Crea**).
2. A cada MV: **Configuració → Xarxa**:
   - **Adaptador 1:** Connectat a **NAT**.
   - **Adaptador 2:** marca **Habilita l'adaptador de xarxa**, Connectat a **Adaptador només amfitrió**, i tria la xarxa de l'apartat anterior.
3. Si cal, canvia el nom de la MV a **Configuració → General → Nom** (`nfs-servidor` i `nfs-client`).

📸 **Captura (`captura-01a.png`):** finestra principal de VirtualBox amb les dues MV a la llista i `nfs-servidor` seleccionada. Al panell de la dreta, a l'apartat **Xarxa**, s'ha de veure `Adaptador 1: ... (NAT)` i `Adaptador 2: ... (Adaptador només amfitrió ...)`.

📸 **Captura (`captura-01b.png`):** el mateix amb `nfs-client` seleccionada.

### 2. Netejar el que quedava de la primera versió (client)

Al client, desmunta la carpeta antiga i **esborra la línia de `/etc/fstab`** que apunta a `192.168.1.153`. Si no ho fas, quan canviï la xarxa el client es pot quedar penjat en arrencar.

```bash
sudo umount /mnt/compartit
sudo nano /etc/fstab
```

Esborra la línia `192.168.1.153:/srv/nfs/compartit /mnt/compartit nfs defaults 0 0` i desa.

### 3. IP estàtica al servidor (netplan)

Engega el servidor i entra-hi per la consola de VirtualBox (encara no té la IP nova).

```bash
ip -br a
ls /etc/netplan/
sudo nano /etc/netplan/50-cloud-init.yaml
```

Deixa el fitxer així (fes servir el nom de fitxer que t'hagi sortit al `ls`):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: false
      addresses:
        - 192.168.56.10/24
```

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
ip -br a
```

**Si falla:** el YAML va amb **espais**, mai tabuladors. Si les interfícies no es diuen `enp0s3` / `enp0s8`, posa els noms que surten a `ip -br a`. Si després de reiniciar es perd la IP, crea `/etc/cloud/cloud.cfg.d/99-disable-network-config.cfg` amb el text `network: {config: disabled}`.

### 4. IP estàtica al client (Zorin OS)

Engega el client i obre un terminal a la sessió gràfica:

```bash
nmcli -t -f DEVICE,CONNECTION device
sudo nmcli con add type ethernet ifname enp0s8 con-name hostonly ipv4.method manual ipv4.addresses 192.168.56.20/24 ipv6.method disabled
sudo nmcli con up hostonly
ip -br a
```

**Si falla:** si `enp0s8` ja tenia una connexió automàtica (p. ex. `Connexió cablejada 2`), desactiva-la: `sudo nmcli con mod "Connexió cablejada 2" connection.autoconnect no` i torna a fer `sudo nmcli con up hostonly`.

### 5. Nom i idioma de cada màquina

Al servidor:

```bash
sudo hostnamectl set-hostname nfs-servidor
sudo sed -i 's/^127.0.1.1.*/127.0.1.1 nfs-servidor/' /etc/hosts
sudo locale-gen ca_ES.UTF-8
sudo update-locale LANG=ca_ES.UTF-8
```

Al client, el mateix canviant el nom per `nfs-client` (o l'idioma des de **Configuració → Regió i idioma**). Tanca la sessió i torna a entrar perquè s'apliqui.

### 6. SSH a les dues màquines

El servidor ja té `openssh-server`. Instal·la'l també al client:

```bash
sudo apt install -y -qq openssh-server
```

Des del PC, obre dues finestres de PowerShell:

```bash
ssh serginovell@192.168.56.10
ssh ⟨usuari del client⟩@192.168.56.20
```

A **cada finestra**:

```bash
clear
hostname
localectl status | grep -E "Locale|Layout|Keymap"
ip -br a
```

📸 **Captura (`captura-02.png`):** les dues finestres una al costat de l'altra. S'ha de veure el nom de cada màquina, l'idioma (`LANG=ca_ES.UTF-8`), `enp0s3` amb `10.0.2.15` i `enp0s8` amb `192.168.56.10` / `192.168.56.20`.

### 7. Comprovar la connectivitat

Al client:

```bash
clear
ping -c 3 192.168.56.10
ping -c 3 google.com
```

📸 **Captura (`captura-03.png`):** els dos pings amb `0% packet loss`: el primer demostra que el client veu el servidor (només amfitrió) i el segon que té internet (NAT).

### 8. Actualitzar els dos sistemes

A **cada finestra**:

```bash
sudo apt update -qq && sudo apt full-upgrade -y -qq
clear
sudo apt update
```

📸 **Captura (`captura-04.png`):** les dues finestres amb l'última línia `Tots els paquets estan actualitzats` (o `All packages are up to date`).

---

## Fase 2. Usuaris, grups i carpetes

### 9. Comprovar que els números estan lliures

A **les dues màquines**. Si no surt res, els UID i GID estan lliures:

```bash
getent group 2001 2002
getent passwd 2101 2102
```

### 10. Crear grups i usuaris (mateix ordre a les dues màquines)

Primer els grups i després els usuaris, sempre amb el número fixat amb `-g` i `-u`. Executa exactament el mateix a **les dues finestres**:

```bash
sudo groupadd -g 2001 devs
sudo groupadd -g 2002 admins
sudo useradd -m -s /bin/bash -u 2101 -g devs dev01
sudo useradd -m -s /bin/bash -u 2102 -g admins admin01
sudo passwd dev01
sudo passwd admin01
clear
getent group devs admins
id dev01
id admin01
```

📸 **Captura (`captura-05.png`):** les dues finestres una al costat de l'altra. Els números han de ser **idèntics** als dos costats: `devs:x:2001`, `admins:x:2002`, `uid=2101(dev01) gid=2001(devs)`, `uid=2102(admin01) gid=2002(admins)`.

### 11. Crear les carpetes amb els permisos (servidor)

```bash
sudo mkdir -p /srv/nfs/devs /srv/nfs/admins
sudo chown root:devs /srv/nfs/devs
sudo chown root:admins /srv/nfs/admins
sudo chmod 2770 /srv/nfs/devs /srv/nfs/admins
clear
ls -ld /srv/nfs/devs /srv/nfs/admins
```

📸 **Captura (`captura-06.png`):** `drwxrws--- root devs /srv/nfs/devs` i `drwxrws--- root admins /srv/nfs/admins`.

**Anàlisi:** apunta per què 2770: el grup té tots els permisos, els altres cap, i la `s` (setgid) fa que els fitxers nous siguin del grup de la carpeta. No fem servir 777 perquè qualsevol usuari hi podria escriure.

---

## Fase 3. Gestió de root_squash

### 12. Paquets NFS

Ja els tens de la primera versió. Si no:

```bash
sudo apt install -y -qq nfs-kernel-server   # servidor
sudo apt install -y -qq nfs-common          # client
```

### 13. Primera exportació (amb root_squash per defecte)

Al servidor, **substitueix tot** el contingut de `/etc/exports` (així també treus la línia antiga de `compartit` amb `*`):

```bash
sudo nano /etc/exports
```

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check)
/srv/nfs/admins  192.168.56.20(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra
clear
grep -v '^#' /etc/exports
sudo exportfs -v
```

📸 **Captura (`captura-07.png`):** les dues línies del fitxer **sense `#` al davant** i l'`exportfs -v` on, entre les opcions, surt **`root_squash`** encara que no l'hem escrit.

### 14. Prova 1: root del client amb root_squash (error)

Al client:

```bash
sudo mkdir -p /mnt/devs /mnt/admins
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
clear
showmount -e 192.168.56.10
sudo touch /mnt/devs/prova_root.txt
sudo ls /mnt/devs
```

📸 **Captura (`captura-08.png`):** el `showmount` amb les dues carpetes i els errors `touch: no s'ha pogut fer touch ... Permís denegat` i `ls: ... Permís denegat`.

**Anàlisi:** el `sudo` del client és root, però el servidor el converteix en `nobody` (UID 65534). `nobody` no és el propietari ni del grup `devs`, i els altres tenen permís `0`, per això no pot fer res.

### 15. Prova 2: amb no_root_squash (funciona)

Al servidor, afegeix `no_root_squash` a la línia de `devs`:

```bash
sudo nano /etc/exports
```

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/admins  192.168.56.20(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra
clear
sudo exportfs -v
```

Al client:

```bash
sudo umount /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
clear
sudo touch /mnt/devs/prova_root.txt
sudo ls -l /mnt/devs
```

📸 **Captura (`captura-09.png`):** les dues finestres. Al servidor, `exportfs -v` amb **`no_root_squash`** a `devs`. Al client, el `touch` sense error i el fitxer `prova_root.txt` amb propietari **`root`**.

**Anàlisi:** ara el root del client continua sent root (UID 0) al servidor i salta els permisos. Compara-ho amb la captura 08: la única diferència és l'opció.

### 16. Tornar a deixar root_squash

Al servidor, treu `no_root_squash` (deixa el fitxer com al pas 13), aplica-ho i esborra el fitxer de prova:

```bash
sudo nano /etc/exports
sudo exportfs -ra
sudo rm /srv/nfs/devs/prova_root.txt
```

---

## Fase 4. Control d'accés per IP i per usuari

### 17. Exportació per IP (RW / RO)

Al servidor, deixa `/etc/exports` així:

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check) 192.168.56.0/24(ro,sync,no_subtree_check)
/srv/nfs/admins  192.168.56.20(rw,sync,no_subtree_check)
```

```bash
sudo nano /etc/exports
sudo exportfs -ra
clear
grep -v '^#' /etc/exports
sudo exportfs -v
```

📸 **Captura (`captura-10.png`):** el fitxer i l'`exportfs -v` on `devs` surt **dues vegades**: `192.168.56.20(...rw...)` i `192.168.56.0/24(...ro...)`, i `admins` només per a `192.168.56.20`.

### 18. Prova per usuari: dev01 contra admin01 (client amb IP .20)

```bash
sudo umount /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/admins /mnt/admins
clear
sudo -u dev01 touch /mnt/devs/fitxer_dev01.txt
sudo -u admin01 touch /mnt/devs/fitxer_admin01.txt
sudo -u admin01 touch /mnt/admins/fitxer_admin01.txt
sudo -u dev01 ls -l /mnt/devs
```

📸 **Captura (`captura-11.png`):** el segon `touch` amb **`Permís denegat`** (admin01 a `devs`), els altres dos sense error, i l'`ls` amb `fitxer_dev01.txt` de propietari `dev01 devs`.

**Anàlisi:** la IP té permís d'escriptura, però NFS després aplica els permisos de Linux pel UID. `admin01` (UID 2102) no és del grup `devs`, per tant no hi pot escriure.

### 19. Prova per IP: canviar el client a 192.168.56.30

```bash
sudo umount /mnt/devs /mnt/admins
sudo nmcli con mod hostonly ipv4.addresses 192.168.56.30/24
sudo nmcli con up hostonly
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
clear
ip -br a show enp0s8
sudo -u dev01 touch /mnt/devs/prova_ip.txt
sudo -u dev01 ls /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/admins /mnt/admins
```

> Si treballes per SSH, en canviar la IP es tallarà la connexió: fes aquest pas des de la consola de Zorin.

📸 **Captura (`captura-12.png`):** `enp0s8` amb `192.168.56.30`, el `touch` amb **`Sistema de fitxers només de lectura`** (Read-only file system), l'`ls` que sí que llista els fitxers, i el `mount` d'`admins` amb **`access denied by server`**.

**Anàlisi:** el mateix usuari `dev01`, que abans podia escriure, ara només pot llegir: la IP `.30` entra per la regla `192.168.56.0/24(ro)`. A `admins` ni tan sols pot muntar, perquè només està exportada a `.20`.

### 20. Tornar a la IP .20

```bash
sudo umount /mnt/devs
sudo nmcli con mod hostonly ipv4.addresses 192.168.56.20/24
sudo nmcli con up hostonly
```

---

## Fase 5. Persistència (fstab)

### 21. Afegir les entrades a /etc/fstab (client)

```bash
sudo nano /etc/fstab
```

Afegeix al final:

```text
192.168.56.10:/srv/nfs/devs    /mnt/devs    nfs  defaults,_netdev  0  0
192.168.56.10:/srv/nfs/admins  /mnt/admins  nfs  defaults,_netdev  0  0
```

```bash
sudo systemctl daemon-reload
sudo mount -a
clear
grep nfs /etc/fstab
df -h | grep /srv/nfs
```

📸 **Captura (`captura-13.png`):** les dues línies de `fstab` i el `df` amb les dues carpetes muntades a `/mnt/devs` i `/mnt/admins`.

**Si falla:** si `mount -a` dona error, revisa que no quedi la línia antiga de `192.168.1.153` (pas 2) i que les rutes siguin exactes.

### 22. Reiniciar i comprovar

```bash
sudo reboot
```

Torna a entrar al client:

```bash
clear
uptime
systemctl --failed
df -h | grep /srv/nfs
sudo -u dev01 touch /mnt/devs/despres_reboot.txt && echo OK
```

📸 **Captura (`captura-14.png`):** l'`uptime` de pocs minuts, `0 loaded units listed` (cap servei fallat), les dues carpetes muntades i `OK`.

---

## Conclusió

No cal captura. Llegeix la conclusió de `documentacio.md` i **reescriu-la amb les teves paraules**: la rúbrica dona 0 punts si sembla text generat sense revisar. Afegeix-hi qualsevol problema que t'hagi sortit a tu.

---

## Resum de captures

| # | Què capturar | Criteri que demostra | Fitxer |
|---|---|---|---|
| 1a | VirtualBox: servidor amb NAT + només amfitrió | 1. Infraestructura | `captura-01a.png` |
| 1b | VirtualBox: client amb NAT + només amfitrió | 1. Infraestructura | `captura-01b.png` |
| 2 | Hostname, idioma i IP de les dues MV | 1. Noms, idiomes i xarxa | `captura-02.png` |
| 3 | Ping al servidor i a internet | 1. Connectivitat | `captura-03.png` |
| 4 | `apt update` sense paquets pendents | 1. Sistemes actualitzats | `captura-04.png` |
| 5 | Grups i usuaris amb UID/GID iguals | 2. Usuaris i grups | `captura-05.png` |
| 6 | `ls -ld` de les carpetes (2770) | 3. Carpetes i permisos | `captura-06.png` |
| 7 | `/etc/exports` i `exportfs -v` amb `root_squash` | 4. root_squash | `captura-07.png` |
| 8 | Root del client: Permís denegat | 4. root_squash (error inicial) | `captura-08.png` |
| 9 | `no_root_squash`: root hi escriu | 4. root_squash (diferència) | `captura-09.png` |
| 10 | `/etc/exports` per IP (RW / RO) | 5. Control d'accés | `captura-10.png` |
| 11 | dev01 escriu, admin01 denegat | 5. Bloqueig per usuari | `captura-11.png` |
| 12 | IP .30: només lectura i access denied | 5. Bloqueig per IP | `captura-12.png` |
| 13 | `fstab` i `mount -a` | 6. Persistència | `captura-13.png` |
| 14 | Després de reiniciar: muntat i sense errors | 6. Persistència | `captura-14.png` |
