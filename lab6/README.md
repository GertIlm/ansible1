# Lab 6: phpMyAdmini paigaldamine ja seadistamine Ansible Playbook'i abil

**Autor:** Gert-Daniel Ilm (ITS24)
**Kursus:** Skriptimine / Halduse automatiseerimine (VOCO)

phpMyAdmini veebipõhine MySQL haldustööriist paigaldatakse ja seadistatakse
Ansible playbookidega kahte Debian 13 kliendiserverisse (grupp `webservers`).
Pärast käivitamist on phpMyAdmin avatav aadressil `http://<kliendi_IP>/phpmyadmin`.

## Keskkond (Proxmox)

| VM | Nimi | Roll | IP |
|----|------|------|----|
| 22340 | gert-lab6-ansible | Ansible juhtserver | 192.168.40.213 |
| 22341 | gert-lab6-web1 | kliendiserver `gert-web1` | 192.168.40.212 |
| 22342 | gert-lab6-web2 | kliendiserver `gert-web2` | 192.168.40.214 |

Masinad on kloonitud Lab 5 masinatest (22310, 22313, 22312), seega on
eeldused juba täidetud: Apache2 + PHP 8.4 (Lab 2-3), MySQL 8.4 root parooliga
failis `/root/.my.cnf` (Lab 4), SSH võtmega ligipääs juhtserverist root
kasutajana ja inventaris grupp `webservers`.

## Failid

```
lab6/
├── ansible.cfg                     # /etc/ansible/ansible.cfg
├── hosts                           # /etc/ansible/hosts (grupp webservers)
├── phpmyadmin.yml                  # põhiülesanne - phpMyAdmini paigaldus
├── phpmyadmin_config.yml           # iseseisev ülesanne 1 - config.inc.php
├── templates/
│   ├── config.inc.php.j2           # Jinja2 mall phpMyAdmini seadetele
│   └── config.header.inc.php.j2    # Jinja2 mall päisele (serveri nimi + IP)
└── pildid/                         # kuvatõmmised
```

Juhtserveris asuvad playbookid kaustas `/root/playbooks/`.

## Käivitamine

```bash
# 1) kontroll (kuivkäivitus) ja päris käivitus - phpMyAdmini paigaldus
ansible-playbook --check /root/playbooks/phpmyadmin.yml
ansible-playbook /root/playbooks/phpmyadmin.yml

# 2) kontroll ja päris käivitus - config.inc.php Jinja2 mallist
ansible-playbook --check /root/playbooks/phpmyadmin_config.yml
ansible-playbook /root/playbooks/phpmyadmin_config.yml
```

## phpmyadmin.yml sammud

Playbook kirjutati samm-sammult: esmalt kaks esimest ülesannet (apt update ja
paigaldus), kontroll `--check` ja käivitus, seejärel lisati ülejäänud sammud.

1. **Uuendame pakettide nimekirja** (`apt: update_cache: yes`)
2. **Eelseadistame phpMyAdmini paigalduse vastused** (`debconf`) - veebiserveriks
   `apache2`, `dbconfig-install: false`, et apt ei jääks interaktiivselt küsima
3. **Paigaldame phpMyAdmin paketi** (`apt: name: phpmyadmin`) - failid kausta `/usr/share/phpmyadmin`
4. **Tagame, et phpMyAdmini sümboolne link on Apache'i kataloogis** (`file: state: link`)
   `/var/www/html/phpmyadmin -> /usr/share/phpmyadmin`
5. **Seome phpMyAdmini Apache konfiguratsiooni** (`file: state: link`)
   `/etc/apache2/conf-available/phpmyadmin.conf -> /etc/phpmyadmin/apache.conf`
6. **Lubame phpMyAdmini Apache konfiguratsiooni** (`command: a2enconf phpmyadmin`, `creates:`)
7. **Taaskäivitame Apache teenuse** (`service: state: restarted`)
8. **Veendume, et Apache töötab ja phpMyAdmin on ligipääsetav** (`uri` + `register` + `failed_when`)
   - HTTP päring `http://localhost/phpmyadmin/`, ebaõnnestub kui staatus pole 200
     või lehel puudub sõna `phpMyAdmin`
9. **phpMyAdmini aadress** (`debug`) - näitab väljundis brauseri aadressi

## Iseseisev ülesanne 1: phpmyadmin_config.yml

`config.inc.php` luuakse Jinja2 mallist `templates/config.inc.php.j2` (põhineb
failil `config.sample.inc.php`) `template` mooduliga. Mallis on muutujad:

- `blowfish_secret` - 32 märgiline juhuslik võti küpsiste krüpteerimiseks.
  Genereeritakse `lookup('password', ...)` abil ja salvestatakse juhtserverisse
  (`/root/playbooks/secrets/`, **ei ole** repos), seega korduval käivitusel
  väärtus ei muutu (idempotentne).
- `$cfg['Servers'][$i]` - MySQL server `localhost:3306`, `auth_type = cookie`
- juurdepääsuseaded - `AllowNoPassword = false`, `AllowRoot = true`
- `TempDir = /var/lib/phpmyadmin/tmp`, `DefaultLang = et`

**Miks `/etc/phpmyadmin/config.inc.php`?** Debiani phpMyAdmini pakett loeb
seadeid failist `/etc/phpmyadmin/config.inc.php`
(`/usr/share/phpmyadmin/libraries/vendor_config.php` -> `configFile`) ning
`/usr/share/phpmyadmin/config.inc.php` faili vaikimisi ei olnud. Seetõttu
kirjutab mall seaded `/etc/phpmyadmin/config.inc.php` faili (root:root, 0644)
ja playbook loob ülesandes toodud asukohta lingi
`/usr/share/phpmyadmin/config.inc.php -> /etc/phpmyadmin/config.inc.php`.
Nii töötab `cat /usr/share/phpmyadmin/config.inc.php` ja phpMyAdmin kasutab
päriselt sama faili.

Sammud:

1. `stat` - kas config.inc.php on olemas (+ `debug` väljund)
2. `copy` (`remote_src`, `force: no`) - kui faili pole, luuakse see `config.sample.inc.php` põhjal
3. `template` - `config.inc.php.j2` -> `/etc/phpmyadmin/config.inc.php` (root:root 0644, `backup: yes`)
4. `file: state: link` - `/usr/share/phpmyadmin/config.inc.php` -> `/etc/phpmyadmin/config.inc.php`
5. `template` - `config.header.inc.php.j2` - phpMyAdmini lehe ülaossa kuvatakse
   `phpMyAdmin @ <hostname> (<IP>)`, et kuvatõmmisel oleks kliendiserveri IP näha
6. `file: state: directory` - TempDir kaust (www-data)
7. `command: php -l` - PHP süntaksi kontroll
8. `uri` - sisselogimisleht vastab (HTTP 200 ja väli `pma_username`)
9. `debug` - kokkuvõte

## Tulemus

- `http://192.168.40.212/phpmyadmin` - gert-web1, phpMyAdmini sisselogimisleht
- `http://192.168.40.214/phpmyadmin` - gert-web2, phpMyAdmini sisselogimisleht

## Kuvatõmmised (pildid/)

| Fail | Sisu |
|------|------|
| 00_samm1_check.png, 00_samm1_kaivitamine.png | 1. samm (apt update + paigaldus): --check ja käivitus |
| 01_playbook_sisu_1.png, 01_playbook_sisu_2.png | phpmyadmin.yml sisu |
| 02_playbook_check.png | phpmyadmin.yml --check |
| 02_playbook_kaivitamine.png | phpmyadmin.yml edukas käivitus |
| 03_config_playbook_sisu_1..3.png | phpmyadmin_config.yml sisu |
| 03_mall_sisu_1.png, 03_mall_sisu_2.png | Jinja2 mallid |
| 04_config_check.png | phpmyadmin_config.yml --check |
| 04_config_kaivitamine_1.png, 04_config_kaivitamine_2.png | phpmyadmin_config.yml edukas käivitus |
| 05_config_inc_php_web1_1.png, 05_config_inc_php_web1_2.png | config.inc.php sisu gert-web1 (192.168.40.212) |
| 05_config_inc_php_web2.png | config.inc.php põhiread gert-web2 (192.168.40.214) |
| 06_brauser_phpmyadmin_web1_212.png, 06_brauser_phpmyadmin_web2_214.png | phpMyAdmin brauseris |
| 07_sisselogimisvaade_web2_214.png | phpMyAdmini sisselogimisvaade (eesti keeles) |
