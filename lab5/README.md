# Lab 5 – WordPressi paigaldamise ja halduse automatiseerimine Ansible Playbook'iga

Autor: Gert-Daniel Ilm (VOCO, IT-süsteemide spetsialist)

## Keskkond (Proxmox)

| VM | Hostname | Roll | IP |
|----|----------|------|----|
| 22310 gert-lab5-ansible | gert-ansible | Ansible-server | 192.168.38.240 |
| 22313 gert-lab5-web1 | gert-web1 | kliendiserver (Apache2 + PHP + MySQL + WordPress) | 192.168.38.242 |
| 22312 gert-lab5-web2 | gert-web2 | kliendiserver (Apache2 + PHP + MySQL + WordPress) | 192.168.38.241 |

Lab 5 masinad on kloonitud Lab 4 masinatest (22300–22302), seega on Lab 4 eeldused juba olemas:
MySQL Community Server 8.4 LTS, root kasutaja parool ja ühenduse andmed failis `/root/.my.cnf`.
Kõik masinad: Debian 13 (ilma GUI-ta), ansible-core 2.19, community.mysql, PHP 8.4.

## Failid

| Fail | Asukoht Ansible-serveris | Kirjeldus |
|------|--------------------------|-----------|
| `ansible.cfg` | `/etc/ansible/ansible.cfg` | Ansible seadistus (inventory, remote_user, host_key_checking) |
| `hosts` | `/etc/ansible/hosts` | inventory, grupp `webservers` (2 kliendiserverit) |
| `wordpress.yml` | `/root/playbooks/wordpress.yml` | WordPressi paigaldus + iseseisvad ülesanded 1 ja 2 |

Käivitamine:

```bash
ansible -m ping webservers
ansible-playbook --check /root/playbooks/wordpress.yml   # kuivkäivitus (kontroll)
ansible-playbook /root/playbooks/wordpress.yml           # päris käivitus
```

## wordpress.yml – sammud

**Play 1: WordPress_Andmebaasi seadistamine**

1. **Paigaldame Apache2 ja PHP** (`apt`) – eeldus Lab 2 ja Lab 3 (apache2, php, php-cli, libapache2-mod-php)
2. **Paigaldame python3-pymysql** (`apt`) – Pythoni moodul, ilma milleta `mysql_db`/`mysql_user` ei tööta
3. **Loome andmebaas nimega-wpdatabase** (`mysql_db`) – root parool loetakse failist `/root/.my.cnf`
4. **Loome kasutaja nimega-wpuser parool-qwerty** (`mysql_user`) – õigused `wpdatabase.*:ALL,GRANT`
5. **Laeme alla WordPressi** (`get_url`) – `https://wordpress.org/latest.tar.gz` → `/tmp/wordpress.tar.gz`
6. **Pakime WordPressi lahti** (`unarchive`, `remote_src: yes`) → `/var/www/html/wordpress`
7. **Kopeerime wp-config-sample.php kui wp-config.php** (`copy`, `force: no`)
8. **Määrame andmebaasi nime / kasutajanime / parooli** (3× `lineinfile`) – `database_name_here`, `username_here`, `password_here` → `wpdatabase`, `wpuser`, `qwerty`
9. **Iseseisev ülesanne 1: Paigaldame PHP MySQL laienduse** (`apt: php-mysql`) – lahendab mysqli vea
10. **Taaskäivita Apache teenus** (`service: state=restarted`)

**Play 2: WordPressi installatsiooni lõpuleviimine (Iseseisev ülesanne 2)**

11. **Laeme alla ja paigaldame wp-cli** (`get_url`) → `/usr/local/bin/wp`, `mode: '0755'`
12. **Kontrollime, kas WordPress on juba paigaldatud** (`command: wp core is-installed`, `register`)
13. **Paigaldame WordPressi** (`command: wp core install … --allow-root`, `args: chdir: /var/www/html/wordpress`)
    – sait „admin wordpressi sait”, aadress `http://<kliendi_IP>/wordpress`, admin kasutaja `admin` / `qwerty`.
    Käivitub ainult siis, kui WordPress pole veel paigaldatud (idempotentne).
14. **WordPressi aadress** (`debug`) – näitab logis saidi ja wp-login.php aadressi

## Miks `--check` annab vea, aga päris käivitus mitte?

`--check` on kuivkäivitus: Ansible ainult **ennustab**, mida muudetaks, aga **ei tee masinas päriselt muudatusi**.

* `get_url` näitab `changed`, kuid faili `/tmp/wordpress.tar.gz` tegelikult alla ei laadita →
  järgmine `unarchive` ülesanne kukub: `Source '/tmp/wordpress.tar.gz' does not exist`.
* Samamoodi `copy` näitab `changed`, kuid `wp-config.php` faili ei looda →
  `lineinfile` kukub: `Destination /var/www/html/wordpress/wp-config.php does not exist !`.

Ülesanded sõltuvad eelmiste ülesannete **tegelikust** tulemusest. Päris käivitusel laaditakse arhiiv alla ja
fail luuakse enne, kui järgmine ülesanne seda vajab, seega vigu ei ole.

## Probleemid ja lahendused

| Probleem | Põhjus | Lahendus |
|----------|--------|----------|
| `Plugin 'mysql_native_password' is not loaded` (mysql_user) | MySQL 8.4 LTS-s on vana `mysql_native_password` plugin vaikimisi välja lülitatud | kasutaja luuakse `plugin: caching_sha2_password` + `plugin_auth_string: qwerty` (+ `update_password: on_create`) |
| Brauseris „Your PHP installation appears to be missing the MySQL extension…” | PHP-l puudub `mysqli` laiendus | **Iseseisev 1:** `apt: name=php-mysql` (Debian 13 → php8.4-mysql; juhendi php7.4-mysql on Debian 11 jaoks) + Apache restart |
| `--check` vead (`unarchive`, `lineinfile`) | kuivkäivitus ei loo faile päriselt | päris käivitus töötab, vt selgitus ülal |
| Hoiatus `column_case_sensitive is not provided` | community.mysql vaikeväärtus muutub | `column_case_sensitive: true` |
| Lab 5 kliendis polnud Apache/PHP-d | masinad kloonitud Lab 4-st (ainult MySQL) | playbooki esimene ülesanne paigaldab apache2 + php |

## Kuvatõmmised (`pildid/`)

| Fail | Sisu |
|------|------|
| `01_playbook_sisu_1..4.png` | wordpress.yml täielik sisu (`cat -n`) |
| `02_playbook_kaivitamine_1..2.png` | edukas `ansible-playbook wordpress.yml` logi |
| `03_ansible_ping.png` | `ansible -m ping webservers` |
| `04_klient_web1_242.png`, `05_klient_web2_241.png` | `/root/.my.cnf` (parool varjatud), wp-config.php DB read, `php -m \| grep mysqli`, `ls -l /var/www/html/wordpress` |
| `06_brauser_mysqli_viga_242.png` | mysqli viga enne iseseisvat ülesannet 1 |
| `07_brauser_install_leht_*.png` | WordPressi installatsioonileht (enne iseseisvat ülesannet 2) |
| `08_brauser_sait_*.png` | töötav WordPressi sait „admin wordpressi sait” |
| `09_brauser_wp_login_242.png` | wp-login.php |
| `10_wpcli_admin_kontroll.png` | wp-cli: admin kasutaja, siteurl, pealkiri, parooli kontroll |
| `11_check_viga_unarchive.png`, `12_check_viga_lineinfile.png` | `--check` vead |
