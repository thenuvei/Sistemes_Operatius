# Activitat NFS: compartició de carpetes entre Ubuntu Server i Zorin OS

Nov 4, 2026 · @Novell

## Introducció

Aquesta activitat comparteix la carpeta `/srv/nfs/compartit` d'un servidor Ubuntu Server amb un client Zorin OS mitjançant NFS, i deixa el muntatge automàtic amb `/etc/fstab`.

NFS (Network File System) és un protocol que permet accedir a fitxers d'un servidor remot com si fossin locals. Identifica els usuaris pel UID i el GID, no pel nom, per això l'usuari ha de tenir els mateixos números al servidor i al client.

| Màquina | Sistema operatiu | Adreça IP | Usuari de prova |
| --- | --- | --- | --- |
| Servidor | Ubuntu Server | (completar) | usuari\_smx (UID 1100, GID 1100) |
| Client | Zorin OS | (completar) | usuari\_smx (UID 1100, GID 1100) |

A cada pas: comanda a dalt, captura al mig i una explicació breu a sota. Substitueix `IP_SERVIDOR` per la IP real del servidor.

## Fase 0: Preparar les màquines i SSH

### Pas 1: Esbrinar la IP de cada màquina

```bash
ip a
```

&#91;Captura: sortida de `ip a` al servidor i al client\]

Consultem la IP de cada màquina. Les dues han d'estar a la mateixa xarxa (NAT o adaptador pont).

### Pas 2: Instal·lar el servidor SSH

```bash
sudo apt update && sudo apt install openssh-server -y
```

&#91;Captura: final de la instal·lació, a cada màquina\]

Instal·lem SSH per poder treballar des del nostre PC amb copiar i enganxar, sense usar la consola de la màquina virtual.

### Pas 3: Connectar per SSH

```bash
ssh usuari@IP_SERVIDOR
ssh usuari@IP_CLIENT
```

&#91;Captura: sessió SSH oberta amb el servidor i amb el client\]

Obrim dues terminals al nostre PC, una per cada màquina. El prompt mostra en quina màquina estem treballant.

### Pas 4: Comprovar la comunicació

```bash
ping -c 3 IP_SERVIDOR
```

&#91;Captura: ping des del client, amb 0% de paquets perduts\]

Verifiquem que el client arriba al servidor abans de configurar res.

## Fase 1: Servidor (Ubuntu Server)

### Pas 5: Instal·lar el servei NFS

```bash
sudo apt update && sudo apt install nfs-kernel-server -y
```

&#91;Captura: final de la instal·lació\]

Instal·lem el paquet que converteix la màquina en servidor NFS.

### Pas 6: Crear el grup i l'usuari amb UID/GID fixos

```bash
sudo groupadd -g 1100 grup_smx
sudo useradd -u 1100 -m -g grup_smx usuari_smx
id usuari_smx
```

&#91;Captura: sortida de `id usuari_smx` amb uid=1100 i gid=1100\]

Creem un usuari amb UID i GID concrets, perquè NFS identifica els usuaris pel número i no pel nom.

### Pas 7: Crear la carpeta i assignar propietari i permisos

```bash
sudo mkdir -p /srv/nfs/compartit
sudo chown usuari_smx:grup_smx /srv/nfs/compartit
sudo chmod 750 /srv/nfs/compartit
ls -ld /srv/nfs/compartit
```

&#91;Captura: `ls -ld` amb drwxr-x--- usuari\_smx grup\_smx\]

La carpeta pertany a l'usuari i al grup creats. El permís 750 dóna accés total al propietari, lectura i execució al grup, i res a la resta.

### Pas 8: Definir l'exportació a /etc/exports

```bash
sudo nano /etc/exports
```

Afegim aquesta línia al final del fitxer:

```text
/srv/nfs/compartit *(rw,sync,no_subtree_check)
```

&#91;Captura: fitxer /etc/exports amb la línia afegida\]

Indiquem quina carpeta es comparteix, amb qui (`*` és qualsevol client) i amb quines opcions: `rw` lectura i escriptura, `sync` escriu al disc abans de respondre, `no_subtree_check` evita comprovacions innecessàries. Per defecte s'aplica `root_squash`, que degrada el root remot a un usuari anònim.

### Pas 9: Aplicar els canvis i verificar el servei

```bash
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
systemctl status nfs-kernel-server
```

&#91;Captura: servei en estat active (running) i sortida de `exportfs -v`\]

Apliquem la configuració, reiniciem el servei i comprovem que funciona i que la carpeta està exportada.

## Fase 2: Client (Zorin OS)

### Pas 10: Instal·lar el client NFS

```bash
sudo apt update && sudo apt install nfs-common -y
```

&#91;Captura: final de la instal·lació\]

Instal·lem les eines necessàries per muntar recursos NFS.

### Pas 11: Crear el mateix grup i usuari

```bash
sudo groupadd -g 1100 grup_smx
sudo useradd -u 1100 -m -g grup_smx usuari_smx -s /bin/bash
sudo passwd usuari_smx
id usuari_smx
```

&#91;Captura: sortida de `id usuari_smx` amb uid=1100 i gid=1100\]

L'usuari ha de tenir el mateix UID i GID que al servidor perquè els permisos de la carpeta coincideixin. Li posem contrasenya per poder iniciar-hi sessió.

### Pas 12: Comprovar que el servidor ofereix el recurs

```bash
showmount -e IP_SERVIDOR
```

&#91;Captura: sortida amb /srv/nfs/compartit \*\]

Verifiquem que el client veu la carpeta exportada pel servidor.

### Pas 13: Crear el punt de muntatge i muntar la carpeta

```bash
sudo mkdir -p /mnt/compartit
sudo mount -t nfs IP_SERVIDOR:/srv/nfs/compartit /mnt/compartit
df -h | grep compartit
```

&#91;Captura: sortida de `df -h` amb el recurs muntat\]

Muntem la carpeta remota en un directori local. A partir d'ara s'hi accedeix com si fos local.

## Fase 3: Proves

### Pas 14: Crear un fitxer com a usuari\_smx (client)

```bash
su - usuari_smx
cd /mnt/compartit
touch fitxer_prova.txt
ls -l fitxer_prova.txt
```

&#91;Captura: fitxer creat, amb propietari usuari\_smx\]

Comprovem que l'usuari pot escriure a la carpeta compartida.

### Pas 15: Verificar el fitxer al servidor

```bash
ls -l /srv/nfs/compartit
```

&#91;Captura: fitxer\_prova.txt visible al servidor\]

Demostrem que el fitxer creat des del client es guarda realment al servidor.

### Pas 16: Provar l'accés amb un altre usuari (opcional)

```bash
exit
ls /mnt/compartit
```

&#91;Captura: error Permission denied\]

Amb un usuari amb un altre UID, el permís 750 denega l'accés. Això confirma que NFS es basa en els UID i GID.

## Fase 4: Muntatge permanent (client)

### Pas 17: Desmuntar la carpeta

```bash
exit
sudo umount /mnt/compartit
df -h | grep compartit
```

&#91;Captura: `df -h` sense cap resultat, perquè ja no està muntada\]

Aquest muntatge era temporal: es perd en desmuntar o en tancar la sessió. Ara el fem automàtic.

### Pas 18: Afegir l'entrada a /etc/fstab

```bash
sudo nano /etc/fstab
```

Afegim aquesta línia al final del fitxer:

```text
IP_SERVIDOR:/srv/nfs/compartit /mnt/compartit nfs defaults 0 0
```

&#91;Captura: fitxer /etc/fstab amb la línia afegida\]

La línia indica el recurs remot (IP i ruta), el punt de muntatge local, el tipus de sistema de fitxers (`nfs`), les opcions (`defaults`) i dos zeros: sense comprovació del sistema de fitxers i sense ordre de muntatge específic.

### Pas 19: Muntar tot el que hi ha a fstab

```bash
sudo mount -a
mount | grep compartit
```

&#91;Captura: sortida de `mount` amb el recurs muntat\]

`mount -a` munta totes les entrades de fstab sense haver de reiniciar.

### Pas 20: Reiniciar i comprovar el muntatge automàtic

```bash
sudo reboot
```

Quan la màquina torni a estar en marxa, ens tornem a connectar per SSH i executem:

```bash
df -h | grep compartit
```

&#91;Captura: el recurs muntat automàticament després de reiniciar\]

Si la carpeta apareix muntada sense fer res, el muntatge és permanent.

### Pas 21: Accedir amb sessió gràfica (opcional)

Iniciem sessió a Zorin OS amb `usuari_smx` i obrim la carpeta `/mnt/compartit`.

&#91;Captura: gestor de fitxers mostrant el contingut de /mnt/compartit\]

Comprovem que l'usuari accedeix a la carpeta compartida de manera automàtica.

## Conclusions

S'ha compartit una carpeta del servidor Ubuntu Server amb un client Zorin OS, i l'usuari `usuari_smx` hi pot crear fitxers perquè té el mateix UID i GID (1100) a les dues màquines.

L'exportació es defineix a `/etc/exports` i el muntatge permanent al client, a `/etc/fstab`. NFS no valida usuaris: es fia dels permisos del servidor, així que cal mantenir els UID i GID coherents (en entorns grans, amb un sistema centralitzat com LDAP).

### Problemes trobats

(Escriu aquí qualsevol problema que hagis tingut i com l'has resolt, o deixa-ho en blanc si no n'hi ha hagut.)
