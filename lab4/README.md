# Lab 4 – Veebiserveri laiendamine MySQL andmebaasiga (Ansible)

Autor: Gert-Daniel Ilm (VOCO, IT-süsteemide spetsialist)

## Keskkond (Proxmox)

| VM | Hostname | Roll | IP |
|----|----------|------|----|
| 22300 gert-lab4-ansible | gert-ansible | Ansible-server | 192.168.37.136 |
| 22301 gert-lab4-web1 | gert-web1 | kliendiserver (MySQL) | 192.168.37.135 |
| 22302 gert-lab4-web2 | gert-web2 | kliendiserver (MySQL) | 192.168.37.137 |

Kõik masinad: Debian 13 (ilma GUI-ta). Ansible-server ühendub klientidesse SSH võtmega root kasutajana.
Ansible 12.0.0 (ansible-core 2.19), community.mysql 3.15.0, MySQL Community Server 8.4.13 LTS.

## Failid

| Fail | Asukoht Ansible-serveris | Kirjeldus |
|------|--------------------------|-----------|
| `ansible.cfg` | `/etc/ansible/ansible.cfg` | Ansible seadistus (inventory, remote_user, host_key_checking) |
| `hosts` | `/etc/ansible/hosts` | inventory, grupp `webservers` |
| `mysql-server.yml` | `/root/playbooks/mysql-server.yml` | MySQL ametliku APT hoidla lisamine ja MySQL serveri paigaldus |
| `mysql-user.yml` | `/root/playbooks/mysql-user.yml` | **Iseseisev ülesanne:** root kasutajale parool `qwerty`, ühenduse andmed faili `/root/.my.cnf` |

## mysql-server.yml – sammud

1. **Loome kataloogi allalaadimiseks** (`file`) – `/root/allalaadimised`, omanik root, õigused 0755
2. **Laadime MySQL APT paketi wget abil** (`shell`) – `mysql-apt-config_0.8.40-1_all.deb` (viimane versioon, toetab Debian 13 trixie)
3. **Paigalda gnupg** (`apt`) – vajalik hoidla GPG-võtmete töötlemiseks
4. **Installeerime mysql repositoorium** (`apt: deb=`) – lisab MySQL hoidla (mysql-8.4-lts) APT allikatesse
5. **Teeme apt-le update** (`apt: update_cache`)
6. **Paigaldame mysql-server** (`apt`)
7. **Veendume, et mysql teenus töötab** (`service`)

Iga samm lisati playbooki ükshaaval, kontrolliti käsuga `ansible-playbook --check` ja seejärel käivitati.

## mysql-user.yml – iseseisev ülesanne

- moodul `community.mysql.mysql_user`: kasutaja `root`@`localhost`, plugin `caching_sha2_password`, parool `qwerty`
  (salt → idempotentne), õigused `*.*:ALL,GRANT`, `update_password: always`
- `check_implicit_admin: yes` – esimesel käivitusel ühendutakse root-iga auth_socket kaudu (ilma paroolita),
  järgmistel käivitustel parooliga
- `/root/.my.cnf` (õigused 0600) – `[client] user=root password=qwerty`, et root saaks käivitada MySQL käske parooli sisestamata
- kontroll playbookis: `mysql --no-defaults -u root` peab andma vea **ERROR 1045 Access denied (using password: NO)**,
  `mysql` (parool failist /root/.my.cnf) peab õnnestuma

## Käivitamine

```bash
ansible-playbook --check /root/playbooks/mysql-server.yml
ansible-playbook /root/playbooks/mysql-server.yml
ansible-playbook /root/playbooks/mysql-user.yml
```

## Kontroll kliendiserveris

```bash
mysql --no-defaults -u root      # ERROR 1045 (28000): Access denied ... (using password: NO)
mysql                            # sisselogimine õnnestub (parool failist /root/.my.cnf)
```
