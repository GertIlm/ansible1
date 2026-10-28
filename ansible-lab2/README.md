# Ansible Lab 2 - Apache2 veebiserveri paigaldamine ja haldus

Autor: Gert-Daniel Ilm (VOCO, Haldustegevuste automatiseerimine)

- apache2.yml - paigaldab Apache2, loob kasutajale user kataloogi public_html, lubab userdir mooduli, kopeerib ja muudab kodulehe (http://server/~user/)
- index.yml - iseseisev ülesanne: unikaalne /var/www/html/index.html (tervitus + hostinimi + IP)
- hosts - inventory (grupp webservers)

Käivitus: ansible-playbook -i hosts apache2.yml ja ansible-playbook -i hosts index.yml
