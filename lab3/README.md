# Lab 3 – Veebiserveri laiendamine PHP toega (Ansible)

Autor: Gert-Daniel Ilm (VOCO, IT-süsteemide spetsialist)

## Keskkond (Proxmox)

| VM | Hostname | Roll | IP |
|----|----------|------|----|
| 22280 gert-lab3-php-ansible | gert-ansible | Ansible-server | 192.168.35.165 |
| 22281 gert-lab3-php-web1 | gert-web1 | kliendiserver (Apache2 + PHP) | 192.168.35.164 |
| 22282 gert-lab3-php-web2 | gert-web2 | kliendiserver (Apache2 + PHP) | 192.168.35.163 |

Kõik masinad: Debian 13 (ilma GUI-ta). Ansible-server ühendub klientidesse SSH võtmega root kasutajana.

## Failid

| Fail | Asukoht Ansible-serveris | Kirjeldus |
|------|--------------------------|-----------|
| `ansible.cfg` | `/etc/ansible/ansible.cfg` | Ansible seadistus (inventory, remote_user) |
| `hosts` | `/etc/ansible/hosts` | inventory, grupp `webservers` |
| `php7.yml` | `/etc/ansible/playbooks/php7.yml` | paigaldab PHP, libapache2-mod-php, php-mysql; loob `phpinfo.php` ja `info.php` (Iseseisev ülesanne 1) |
| `uninst_php.yml` | `/etc/ansible/playbooks/uninst_php.yml` | eemaldab PHP versiooni koos seotud pakettidega (Iseseisev ülesanne 2) |

## Kasutamine

```bash
# kontroll (midagi ei muudeta)
ansible-playbook --check /etc/ansible/playbooks/php7.yml
# PHP paigaldus + testfailid
ansible-playbook /etc/ansible/playbooks/php7.yml
# PHP eemaldamine
ansible-playbook /etc/ansible/playbooks/uninst_php.yml
# PHP taastamine
ansible-playbook /etc/ansible/playbooks/php7.yml
```

Kontroll brauseris: `http://<kliendi_serveri_ip>/info.php` või `http://<kliendi_serveri_ip>/phpinfo.php`.

NB! Debian 13 repos ei ole php7.x pakette, seega paigaldatakse üldine pakett `php` (`state=latest` → PHP 8.4).
