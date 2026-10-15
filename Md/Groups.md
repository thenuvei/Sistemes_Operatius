# Activitat: Gestió d’usuaris i grups a Ubuntu Server

**Alumne:** Sergi Novell  
**Curs:** SMX  
**Data:** 15/10/2026  

En aquesta activitat he creat grups i usuaris, he personalitzat els comptes nous, he bloquejat i eliminat usuaris i he canviat la shell d’un usuari.

---

## 1. Creació dels grups

Primer he creat els tres grups que demanava l’activitat.

```bash
sudo addgroup devs
```

![Creació del grup devs](img/01_addgroup_devs.png)

```bash
sudo addgroup sysadmin
```

![Creació del grup sysadmin](img/02_addgroup_sysadmin.png)

```bash
sudo addgroup auditor
```

![Creació del grup auditor](img/03_addgroup_auditor.png)

Després he comprovat que s’hagin creat correctament.

```bash
tail -n 10 /etc/group
```

![Comprovació dels grups a /etc/group](img/04_tail_etc_group.png)

```bash
getent group devs sysadmin auditor
```

![GID dels tres grups](img/05_getent_grups.png)

**Conclusió:** els grups s’han creat correctament i cada un té el seu GID.

---

## 2. Personalització dels usuaris nous

He modificat la carpeta `/etc/skel`, que és la plantilla dels usuaris nous.

### Fitxer de benvinguda

```bash
sudo nano /etc/skel/BENVINGUDA.txt
```

![Fitxer de benvinguda](img/06_nano_benvinguda.png)

He posat aquest contingut:

```text
Benvingut/da al servidor de l'empresa.

Normes d'ús:
- No comparteixis les teves credencials.
- Utilitza contrasenyes segures.
- No instal·lis programari sense autorització.
- Informa l'administració de qualsevol incidència.
```

### Carpeta Documents

```bash
sudo mkdir -p /etc/skel/Documents
```

![Creació de la carpeta Documents](img/07_mkdir_documents.png)

### Alias `ll`

```bash
sudo nano /etc/skel/.bashrc
```

![Alias ll al .bashrc](img/08_nano_bashrc.png)

He afegit aquesta línia:

```bash
alias ll='ls -la --color=auto'
```

### Comprovació de la plantilla

```bash
ls -la /etc/skel
```

![Contingut de /etc/skel](img/09_ls_skel.png)

```bash
tail -n 5 /etc/skel/.bashrc
```

![Comprovació de l’alias](img/10_tail_bashrc.png)

### Shell per defecte

```bash
sudo nano /etc/adduser.conf
```

![Configuració de DSHELL](img/11_nano_adduser_conf.png)

He comprovat que estigui ben configurat:

```bash
grep DSHELL /etc/adduser.conf
```

![Comprovació de DSHELL](img/12_grep_dshell.png)

---

## 3. Prova amb un usuari nou

He creat un usuari de prova per comprovar que la plantilla funciona.

```bash
sudo adduser provaXX
```

![Creació de l’usuari de prova](img/13_adduser_prova.png)

He entrat amb l’usuari de prova:

```bash
su - provaXX
```

![Inici de sessió de provaXX](img/14_su_prova.png)

```bash
ls -la
```

![Contingut del home de provaXX](img/15_ls_home_prova.png)

```bash
cat BENVINGUDA.txt
```

![Fitxer de benvinguda de provaXX](img/16_cat_benvinguda.png)

```bash
type ll
```

![Comprovació de l’alias ll](img/17_type_ll.png)

```bash
ll
```

![Funcionament de l’alias ll](img/18_ll.png)

He sortit de l’usuari de prova:

```bash
exit
```

![Sortida de la sessió de prova](img/19_exit_prova.png)

Finalment he eliminat l’usuari de prova:

```bash
sudo deluser --remove-home provaXX
```

![Eliminació de l’usuari de prova](img/20_deluser_prova.png)

---

## 4. Creació dels usuaris definitius

### Usuaris `pau_dev` i `laura_dev`

```bash
sudo adduser --ingroup devs pau_dev
```

![Creació de pau_dev](img/21_adduser_pau_dev.png)

```bash
sudo adduser --ingroup devs laura_dev
```

![Creació de laura_dev](img/22_adduser_laura_dev.png)

### Usuari `marc_sys`

```bash
sudo useradd -m -d /home/marc_sys -s /bin/bash -g sysadmin -G devs -c "Marc Soler" marc_sys
```

![Creació de marc_sys](img/23_useradd_marc_sys.png)

```bash
sudo passwd marc_sys
```

![Contrasenya de marc_sys](img/24_passwd_marc_sys.png)

### Usuari `auditor`

```bash
sudo useradd -m -d /home/auditor -s /bin/bash -g auditor -c "Usuari Auditor" auditor
```

![Creació d’auditor](img/25_useradd_auditor.png)

```bash
sudo passwd auditor
```

![Contrasenya d’auditor](img/26_passwd_auditor.png)

### Comprovació dels usuaris

```bash
id pau_dev
```

![Informació de pau_dev](img/27_id_pau_dev.png)

```bash
id laura_dev
```

![Informació de laura_dev](img/28_id_laura_dev.png)

```bash
id marc_sys
```

![Informació de marc_sys](img/29_id_marc_sys.png)

```bash
id auditor
```

![Informació d’auditor](img/30_id_auditor.png)

### Obligar `pau_dev` a canviar la contrasenya

```bash
sudo chage -d 0 pau_dev
```

![Forçar canvi de contrasenya](img/31_chage_pau_dev.png)

```bash
sudo chage -l pau_dev
```

![Comprovació de chage](img/32_chage_l_pau_dev.png)

```bash
su - pau_dev
```

![pau_dev obligat a canviar la contrasenya](img/33_su_pau_dev_chage.png)

---

## 5. Inspecció del sistema

```bash
id marc_sys
```

![UID, GID i grups de marc_sys](img/34_id_marc_sys_inspeccio.png)

```bash
grep '^pau_dev:' /etc/passwd
```

![Informació de pau_dev a /etc/passwd](img/35_grep_passwd_pau.png)

```bash
sudo grep '^pau_dev:' /etc/shadow
```

![Informació de pau_dev a /etc/shadow](img/36_grep_shadow_pau.png)

```bash
ls -l /etc/shadow
```

![Permisos de /etc/shadow](img/37_ls_shadow.png)

```bash
ls -la /etc/skel
```

![Directori plantilla /etc/skel](img/38_ls_skel_inspeccio.png)

```bash
grep -E '^(DSHELL|SKEL)' /etc/adduser.conf
```

![Configuració d’adduser.conf](img/39_grep_adduser_conf.png)

**Respostes:**

- `marc_sys` té el grup principal `sysadmin` i també pertany al grup `devs`.
- `/etc/passwd` conté informació bàsica de l’usuari, mentre que `/etc/shadow` conté la contrasenya xifrada i les dates de caducitat.
- `/etc/shadow` té permisos restringits perquè els usuaris normals no puguin veure les contrasenyes xifrades.
- El directori plantilla és `/etc/skel` i està definit a `/etc/adduser.conf`.

---

## 6. Bloqueig i eliminació d’usuaris

### Bloquejar `pau_dev`

```bash
sudo usermod -L pau_dev
```

![Bloqueig de pau_dev](img/40_usermod_L_pau.png)

```bash
sudo grep '^pau_dev:' /etc/shadow
```

![Compte bloquejat a /etc/shadow](img/41_shadow_pau_bloquejat.png)

```bash
sudo passwd -S pau_dev
```

![Estat bloquejat de pau_dev](img/42_passwd_S_pau_bloquejat.png)

### Desbloquejar `pau_dev`

```bash
sudo usermod -U pau_dev
```

![Desbloqueig de pau_dev](img/43_usermod_U_pau.png)

```bash
sudo passwd -S pau_dev
```

![Compte desbloquejat](img/44_passwd_S_pau_desbloquejat.png)

### Eliminar `auditor`

```bash
sudo deluser --remove-home --remove-all-files auditor
```

![Eliminació d’auditor](img/45_deluser_auditor.png)

```bash
id auditor
```

![Comprovació que auditor no existeix](img/46_id_auditor_eliminat.png)

```bash
ls -la /home
```

![Home sense la carpeta d’auditor](img/47_ls_home_sense_auditor.png)

---

## 7. Canvi de shell de `laura_dev`

Primer he instal·lat fish.

```bash
sudo apt update
```

![Actualització de paquets](img/48_apt_update.png)

```bash
sudo apt install fish -y
```

![Instal·lació de fish](img/49_apt_install_fish.png)

```bash
command -v fish
```

![Ruta d’instal·lació de fish](img/50_command_v_fish.png)

```bash
cat /etc/shells
```

![Shells disponibles](img/51_cat_etc_shells.png)

Si no apareix fish, l’he afegit:

```bash
command -v fish | sudo tee -a /etc/shells
```

![Afegir fish a /etc/shells](img/52_afegir_fish_shells.png)

He canviat la shell de `laura_dev`:

```bash
sudo usermod -s /usr/bin/fish laura_dev
```

![Canvi de shell de laura_dev](img/53_usermod_fish_laura.png)

He comprovat el canvi:

```bash
grep '^laura_dev:' /etc/passwd
```

![Shell de laura_dev a /etc/passwd](img/54_grep_laura_passwd.png)

He entrat amb l’usuari:

```bash
su - laura_dev
```

![Sessió de laura_dev amb fish](img/55_su_laura_fish.png)

Dins de fish he comprovat la shell:

```bash
echo $SHELL
```

![Comprovació de la variable SHELL](img/56_echo_shell.png)

```bash
ps -p $$
```

![Procés actual amb fish](img/57_ps_fish.png)

He sortit de la sessió:

```bash
exit
```

![Sortida de la sessió de laura_dev](img/58_exit_laura.png)

---

## Conclusió

He aconseguit crear els grups i usuaris demanats, personalitzar els comptes nous amb `/etc/skel`, bloquejar i eliminar usuaris i canviar la shell de `laura_dev` a fish. Totes les comprovacions han funcionat correctament.