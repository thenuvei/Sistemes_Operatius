# Pràctica NFS: carpetes compartides amb control d'accés entre Ubuntu Server i Zorin OS

**Alumne:** Sergi Novell
**Servei:** NFS (`nfs-kernel-server` / `nfs-common`)
**Sistema:** Ubuntu Server (`nfs-servidor`) + Zorin OS (`nfs-client`) en VirtualBox
**Data:** ⟨data⟩

---

## Escenari

| Màquina | Sistema | Adaptador 1 (NAT) | Adaptador 2 (només amfitrió) | Idioma |
|---|---|---|---|---|
| `nfs-servidor` | Ubuntu Server | DHCP, `⟨10.0.2.15⟩` | `192.168.56.10/24` | `ca_ES.UTF-8` |
| `nfs-client` | Zorin OS | DHCP, `⟨10.0.2.15⟩` | `192.168.56.20/24` | `ca_ES.UTF-8` |

L'adaptador NAT dona internet a cada màquina i l'adaptador només amfitrió les connecta entre elles i amb el PC, des d'on treballo per SSH.

| Grup / usuari | GID / UID | Carpeta | Permisos |
|---|---|---|---|
| `devs` | 2001 | `/srv/nfs/devs` | `root:devs` 2770 |
| `admins` | 2002 | `/srv/nfs/admins` | `root:admins` 2770 |
| `dev01` (grup `devs`) | 2101 | | |
| `admin01` (grup `admins`) | 2102 | | |

---

## Fase 1. Infraestructura i xarxa

### 1. Xarxa a VirtualBox

![Servidor amb adaptador NAT i adaptador només amfitrió](./img/captura-01a.png)
![Client amb adaptador NAT i adaptador només amfitrió](./img/captura-01b.png)

A cada MV: **Configuració → Xarxa**. L'adaptador 1 en NAT i l'adaptador 2 en **Adaptador només amfitrió** (xarxa `192.168.56.0/24`). Les dues MV tenen el nom `nfs-servidor` i `nfs-client`.

### 2. Neteja de la primera versió

```bash
sudo umount /mnt/compartit
sudo nano /etc/fstab
```

Al client he esborrat la línia de `fstab` que muntava `192.168.1.153:/srv/nfs/compartit`, perquè amb la xarxa nova aquesta IP ja no existeix i podria bloquejar l'arrencada.

### 3. IP estàtica al servidor

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
sudo nano /etc/netplan/50-cloud-init.yaml
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

`enp0s3` (NAT) agafa la IP per DHCP i `enp0s8` (només amfitrió) té la IP fixa del servidor.

### 4. IP estàtica al client

```bash
sudo nmcli con add type ethernet ifname enp0s8 con-name hostonly ipv4.method manual ipv4.addresses 192.168.56.20/24 ipv6.method disabled
sudo nmcli con up hostonly
```

Zorin fa servir NetworkManager, per això la IP es posa amb `nmcli` i no amb netplan.

### 5. Nom, idioma i SSH

![Hostname, idioma i IP de les dues màquines](./img/captura-02.png)

```bash
sudo hostnamectl set-hostname nfs-servidor      # nfs-client al client
sudo locale-gen ca_ES.UTF-8
sudo update-locale LANG=ca_ES.UTF-8
sudo apt install -y openssh-server               # al client
hostname
localectl status | grep -E "Locale|Layout|Keymap"
ip -br a
```

Cada màquina té el seu nom i l'idioma en català. Les dues tenen `enp0s3` amb la IP del NAT i `enp0s8` amb la IP fixa. Treballo per SSH des del PC amb una finestra per màquina.

### 6. Connectivitat

![Ping del client al servidor i a internet](./img/captura-03.png)

```bash
ping -c 3 192.168.56.10
ping -c 3 google.com
```

El client arriba al servidor per la xarxa només amfitrió i a internet pel NAT, amb `0% packet loss` als dos.

### 7. Actualització dels sistemes

![Les dues màquines sense paquets pendents](./img/captura-04.png)

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt update
```

Després d'actualitzar, el segon `apt update` diu que tots els paquets estan actualitzats a les dues màquines.

---

## Fase 2. Usuaris i grups

### 8. Creació amb UID i GID fixos

![Grups i usuaris amb els mateixos números al servidor i al client](./img/captura-05.png)

```bash
getent group 2001 2002          # comprovo que els números estan lliures
getent passwd 2101 2102
sudo groupadd -g 2001 devs
sudo groupadd -g 2002 admins
sudo useradd -m -s /bin/bash -u 2101 -g devs dev01
sudo useradd -m -s /bin/bash -u 2102 -g admins admin01
sudo passwd dev01
sudo passwd admin01
getent group devs admins
id dev01
id admin01
```

He executat les mateixes comandes i en el mateix ordre a les dues màquines: primer els grups i després els usuaris, amb el número fixat amb `-g` i `-u`.

**Resultat:**
- `devs` = 2001 i `admins` = 2002 al servidor i al client.
- `dev01` = 2101 i `admin01` = 2102 als dos costats.
- Ha de coincidir perquè NFS no envia el nom de l'usuari, sinó el número. Si al client `dev01` fos 1001, el servidor el tractaria com un altre usuari.

---

## Fase 2. Carpetes i permisos locals

### 9. Estructura de carpetes

![Carpetes devs i admins amb permisos 2770](./img/captura-06.png)

```bash
sudo mkdir -p /srv/nfs/devs /srv/nfs/admins
sudo chown root:devs /srv/nfs/devs
sudo chown root:admins /srv/nfs/admins
sudo chmod 2770 /srv/nfs/devs /srv/nfs/admins
ls -ld /srv/nfs/devs /srv/nfs/admins
```

Cada carpeta és del grup que hi ha de treballar.

**Justificació dels permisos (`drwxrws---`):**
- `7` per al propietari i per al grup: els membres de `devs` o `admins` poden llegir i escriure a la seva carpeta.
- `0` per als altres: un usuari que no és del grup no hi pot ni entrar.
- `2` (setgid, la `s`): els fitxers nous agafen el grup de la carpeta i no el de l'usuari, així tots els membres del grup els poden editar.
- No faig servir 777 perquè llavors qualsevol usuari, de qualsevol grup, hi podria escriure.

---

## Fase 3. Gestió de root_squash

### 10. Exportació inicial

![/etc/exports i exportfs -v amb root_squash](./img/captura-07.png)

```bash
sudo apt install -y nfs-kernel-server     # servidor
sudo apt install -y nfs-common            # client
sudo nano /etc/exports
```

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check)
/srv/nfs/admins  192.168.56.20(rw,sync,no_subtree_check)
```

```bash
sudo exportfs -ra
grep -v '^#' /etc/exports
sudo exportfs -v
```

Les dues carpetes només s'exporten al client (`192.168.56.20`). A `exportfs -v` surt `root_squash` encara que no l'he escrit, perquè és l'opció per defecte.

### 11. Prova 1: root del client amb root_squash

![El root del client rep Permís denegat](./img/captura-08.png)

```bash
sudo mkdir -p /mnt/devs /mnt/admins
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
showmount -e 192.168.56.10
sudo touch /mnt/devs/prova_root.txt
sudo ls /mnt/devs
```

El recurs es munta bé, però el root del client no hi pot crear fitxers ni llistar-los: `Permís denegat`.

### 12. Prova 2: amb no_root_squash

![Amb no_root_squash el root del client hi escriu](./img/captura-09.png)

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo exportfs -v
sudo umount /mnt/devs                                    # client
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs  # client
sudo touch /mnt/devs/prova_root.txt
sudo ls -l /mnt/devs
```

Amb `no_root_squash` el mateix `touch` funciona i el fitxer queda amb propietari `root`.

**Què és root_squash i per què passa:**
- Per defecte NFS fa **root_squash**: quan una petició arriba del root del client (UID 0), el servidor la tracta com si vingués de `nobody` (UID 65534).
- A la prova 1, `nobody` no és el propietari de `/srv/nfs/devs` ni del grup `devs`, i els altres tenen permís `0`, per això rep `Permís denegat`.
- A la prova 2, amb `no_root_squash`, el root del client continua sent UID 0 al servidor i salta tots els permisos.
- Serveix perquè ser administrador d'un client no et faci administrador dels fitxers del servidor. Per això he tornat a deixar `root_squash`:

```bash
sudo nano /etc/exports      # trec no_root_squash
sudo exportfs -ra
sudo rm /srv/nfs/devs/prova_root.txt
```

---

## Fase 4. Control d'accés per IP i per usuari

### 13. Exportació per IP (RW / RO)

![/etc/exports amb regles diferents per IP](./img/captura-10.png)

```text
/srv/nfs/devs    192.168.56.20(rw,sync,no_subtree_check) 192.168.56.0/24(ro,sync,no_subtree_check)
/srv/nfs/admins  192.168.56.20(rw,sync,no_subtree_check)
```

```bash
sudo nano /etc/exports
sudo exportfs -ra
grep -v '^#' /etc/exports
sudo exportfs -v
```

| Recurs | `192.168.56.20` | Resta de `192.168.56.0/24` |
|---|---|---|
| `/srv/nfs/devs` | lectura i escriptura (`rw`) | només lectura (`ro`) |
| `/srv/nfs/admins` | lectura i escriptura (`rw`) | sense accés |

Quan una IP encaixa en més d'una regla, NFS fa servir la més concreta: `.20` agafa la seva línia `rw` i no la de la xarxa.

### 14. Bloqueig per permisos d'usuari (dev01 contra admin01)

![dev01 escriu a devs i admin01 rep Permís denegat](./img/captura-11.png)

```bash
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/admins /mnt/admins
sudo -u dev01 touch /mnt/devs/fitxer_dev01.txt
sudo -u admin01 touch /mnt/devs/fitxer_admin01.txt
sudo -u admin01 touch /mnt/admins/fitxer_admin01.txt
sudo -u dev01 ls -l /mnt/devs
```

Des de la mateixa IP `.20`, que té `rw`, `dev01` pot escriure a `devs` però `admin01` rep `Permís denegat`. En canvi, `admin01` sí que pot escriure a `admins`.

**Resultat:** aquí el bloqueig no és de NFS, és de Linux. L'export deixa escriure la IP, però després el servidor comprova el UID: `admin01` (2102) no és del grup `devs` i la carpeta no dona permisos als altres.

### 15. Bloqueig per IP (client amb 192.168.56.30)

![Amb la IP .30 devs és només de lectura i admins es denega](./img/captura-12.png)

```bash
sudo umount /mnt/devs /mnt/admins
sudo nmcli con mod hostonly ipv4.addresses 192.168.56.30/24
sudo nmcli con up hostonly
sudo mount -t nfs 192.168.56.10:/srv/nfs/devs /mnt/devs
ip -br a show enp0s8
sudo -u dev01 touch /mnt/devs/prova_ip.txt
sudo -u dev01 ls /mnt/devs
sudo mount -t nfs 192.168.56.10:/srv/nfs/admins /mnt/admins
```

He canviat la IP del client a `.30`. El mateix `dev01` ara rep `Sistema de fitxers només de lectura` en escriure, però pot llegir els fitxers. El muntatge d'`admins` falla amb `access denied by server`.

**Resultat:** aquest bloqueig sí que és de NFS: la IP `.30` entra per la regla `192.168.56.0/24(ro)` de `devs` i no té cap regla a `admins`. Després he tornat a posar la IP `.20`:

```bash
sudo umount /mnt/devs
sudo nmcli con mod hostonly ipv4.addresses 192.168.56.20/24
sudo nmcli con up hostonly
```

---

## Fase 5. Persistència (fstab)

### 16. Entrades a /etc/fstab

![fstab amb les dues carpetes i mount -a](./img/captura-13.png)

```text
192.168.56.10:/srv/nfs/devs    /mnt/devs    nfs  defaults,_netdev  0  0
192.168.56.10:/srv/nfs/admins  /mnt/admins  nfs  defaults,_netdev  0  0
```

```bash
sudo nano /etc/fstab
sudo systemctl daemon-reload
sudo mount -a
grep nfs /etc/fstab
df -h | grep /srv/nfs
```

Cada línia indica el recurs del servidor, el punt de muntatge, el tipus `nfs` i les opcions. `_netdev` fa que s'esperi a tenir xarxa abans de muntar. `mount -a` munta tot el que hi ha a `fstab` sense reiniciar.

### 17. Comprovació després de reiniciar

![Després de reiniciar les carpetes estan muntades i no hi ha errors](./img/captura-14.png)

```bash
sudo reboot
uptime
systemctl --failed
df -h | grep /srv/nfs
sudo -u dev01 touch /mnt/devs/despres_reboot.txt && echo OK
```

Després de reiniciar, el sistema arrenca sense serveis fallats (`0 loaded units listed`), les dues carpetes es munten soles i `dev01` hi pot escriure.

---

## Problemes trobats

### L'exportació estava comentada

En una primera versió de la pràctica (adaptador pont i servidor a `192.168.1.153`), el client no podia muntar la carpeta.

![/etc/exports amb la línia comentada](./img/08_servidor_exports_error.png)
![showmount amb la llista buida](./img/12_client_showmount_buit.png)
![mount.nfs: access denied by server](./img/13_client_mount_denegat.png)

- **Error:** `showmount -e` sortia buit i el `mount` donava `access denied by server while mounting`.
- **Causa:** la línia `/srv/nfs/compartit *(rw,sync,no_subtree_check)` de `/etc/exports` tenia un `#` al davant. El servei estava `active`, però no exportava res.
- **Solució:** treure el `#` i fer `sudo exportfs -ra`. Des de llavors comprovo sempre amb `exportfs -v` abans de muntar res.

### Altres problemes

| Problema | Causa | Solució |
|---|---|---|
| El primer `mount` es va quedar penjat i el vaig aturar amb `Ctrl` + `C` | Havia copiat la comanda amb `IP_SERVIDOR` sense canviar-la per la IP real | Posar la IP del servidor |
| `passwd` no acceptava la contrasenya | Era massa curta o massa simple | Posar-ne una de més llarga |
| La primera versió feia servir adaptador pont i `*` a `exports` | No complia la rúbrica (NAT + només amfitrió i control per IP) | Canviar la xarxa, esborrar l'entrada antiga de `fstab` i exportar per IP |
| ⟨altres problemes que t'hagin sortit⟩ | ⟨causa⟩ | ⟨solució⟩ |

---

## Conclusió

> ✏️ **Reescriu aquest apartat amb les teves paraules abans d'entregar-lo.**

La pràctica m'ha servit per veure que NFS és molt fàcil de posar en marxa però que la seguretat depèn de com el configures. L'error que més temps em va fer perdre va ser una línia comentada a `/etc/exports`: el servei estava `active` i no donava cap error, simplement no exportava res. D'aquí he après a comprovar sempre amb `exportfs -v` i `showmount -e` abans de muntar res al client.

**Limitacions que he vist:**
- **NFS es refia del client.** El servidor no comprova contrasenyes, només el UID que li envia el client. Si algú té root en una màquina amb IP permesa, pot crear un usuari amb UID 2101 i el servidor el tractarà com a `dev01`. El `root_squash` només protegeix contra el root, no contra això.
- **La IP es pot canviar.** A la prova de la fase 4 he canviat la IP del client amb una sola comanda. Igual que jo he passat de `.20` a `.30`, algú altre es podria posar la `.20` i tenir accés d'escriptura.
- **El trànsit no va xifrat.** Tot el que es copia a les carpetes passa en clar per la xarxa.
- **Els usuaris s'han de crear a mà a cada màquina.** Amb dos usuaris és fàcil, però en una empresa amb molts equips és molt fàcil equivocar-se de UID.

**Què milloraria:**
- Fer servir **Kerberos** (`sec=krb5p`): l'usuari s'ha d'autenticar de debò i, a més, el trànsit va xifrat. Soluciona les tres primeres limitacions.
- Tenir els usuaris en un **servidor LDAP** (o FreeIPA) perquè els UID i GID siguin iguals a tots els equips sense crear-los a mà.
- Activar el **tallafocs** (`ufw`) al servidor i deixar passar el port 2049 només des de `192.168.56.20`, perquè la restricció no depengui només d'`/etc/exports`.
- No fer servir mai `no_root_squash` fora de proves, i per a carpetes públiques fer servir `all_squash` perquè tothom entri com a `nobody`.
