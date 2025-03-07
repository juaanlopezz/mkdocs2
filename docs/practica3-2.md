# Practica-IAW-3.1- Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

## En esta práctica vamos a realizar la implantación de la aplicación web Moodle en dos instancias EC2 de Amazon Web Services (AWS) haciendo uso de playbooks de Ansible. En una de las instancias deberá instalar Apache HTTP Server y los módulos necesarios de PHP y en la otra máquina deberá instalar MySQL Server.

## Implantación de Moodle en Amazon Web Services (AWS) mediante Ansible

## Pasos a seguir antes de todo:

## Tener la siguiente estructura de directorios y archivos dentro de la instancia del nodo control, ya que todo lo que vamos a hacer va a ser desde el nodo control:

![](imagenes/estructura.png)

## La arquitectura de esta aplicación estará formada por tres capas:

### - Una capa con un balanceador de carga que repartirá las peticiones entre los servidores web.
### - Una capa de front-end, formada por un servidor web con Apache HTTP Server.
### - Una capa de back-end, formada por un servidor MySQL.
## 


## 1. paso: Creamos el nombre de dominio desde no-ip, poniendo la ip del frontend
![](imagenes/no-ip.png)

## 2. paso:  Ahora vamos a crear el archivo de inventory, donde en inventory tenemos que poner las ip pública del frontend, backend, nfs, y del loadbalancer.
### hice otro grupo para frontend 1 para solo instalar wordpress en un frontend 
```yaml
[frontend]
52.22.98.141
44.199.57.156

[frontend1]
52.22.98.141

[backend]
35.168.180.126

[load_balancer]
98.85.8.55

[nfs]
54.84.33.233

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=/home/ubuntu/practica-iaw-3.1/vockey.pem
ansible_ssh_common_args='-o StrictHostKeyChecking=accept-new'
```

## 3. paso: vamos a utilizar las siguientes variables donde se definiran dentro de su arhcivo correspondiente:
```yaml
# Variables de configuración de Let's Encrypt
LE_EMAIL: "juanje@gmail.com"
LE_DOMAIN: "practica-iaw.zapto.org"

# Variables de MySQL
mysql_user: "juanje"
mysql_password: "eliker123"
mysql_host: "172.31.85.101"

# Variables de configuración de la base de datos de WordPress
WORDPRESS_DB_NAME: "wordpress"
WORDPRESS_DB_USER: "juanje"
WORDPRESS_DB_PASSWORD: "eliker123"
WORDPRESS_DB_HOST: "172.31.85.101"
WORDPRESS_DIRECTORY: "/var/www/html"
wordpress_locale: "es_ES"



# Variables para la configuración del sitio de WordPress
locale: "es_ES"  # Localización de WordPress
wordpress_title: "Mi WordPress juanje"  # Título del sitio de WordPress
wordpress_admin_user: "admin"  # Usuario del administrador
wordpress_admin_password: "admin"  # Contraseña del administrador
wordpress_admin_email: "admin@example.com"  # Correo del administrador
wordpress_hide_login_url: "nada"  # URL oculta para el login
domain: "https://practica-iaw.zapto.org"

# IPs y configuraciones de red
BACKEND_PRIVATE_IP: "172.31.85.101"
IP_FRONTEND_1: "172.31.24.92"
IP_FRONTEND_2: "172.31.23.88"
SERVER_IP: "172.31.28.85"
CLIENT_IP: "172.31.24.92"
FRONTEND_PRIVATE_IP: "172.31.%"
FRONTEND_IP_RANGE: "172.31.0.0/24"


```

## 4. paso: crear el archivo install_lamp_frontend.yml
```yaml
---
- name: Instalar LAMP en Frontend
  hosts: frontend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:

    # Actualizamos los repositorios para tener las últimas versiones de los paquetes
    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    # Instalamos Apache
    - name: Instalar el servidor web Apache
      apt:
        name: apache2
        state: present

    # Habilitamos el módulo rewrite de Apache para permitir la reescritura de URLs
    - name: Habilitar el módulo rewrite de Apache
      apache2_module:
        name: rewrite
        state: present

    # Copiamos el archivo de configuración de Apache desde mis plantillas
    - name: Copiar el archivo de configuración de Apache
      copy:
        src: ../templates/000-default.conf
        dest: /etc/apache2/sites-available/
        mode: 0755

    # Instalamos PHP y los módulos necesarios para ejecutar WordPress o Moodle
    - name: Instalar PHP y los módulos necesarios
      apt: 
        name: 
          - php
          - libapache2-mod-php
          - php-mysql
          - php-xml
          - php-mbstring
          - php-curl
          - php-zip
          - php-gd
          - php-intl
          - php-soap
          - php-ldap
          - php-opcache
          - php-cli
        state: present

    # Reiniciamos Apache para aplicar los cambios de configuración
    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
        
```

## 5. paso: crear el archivo install_lamp_backend.yml
```yaml
---
- name: Instalar MySQL en Backend
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:
    - name: Actualizar repositorios
      ansible.builtin.apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Instalar MySQL Server
      apt:
        name: mysql-server
        state: present

    - name: Configurar MySQL para aceptar conexiones remotas
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^bind-address'
        line: "bind-address = {{ BACKEND_PRIVATE_IP }}"
      notify: Reiniciar MySQL

  handlers:
    - name: Reiniciar MySQL
      service:
        name: mysql
        state: restarted

```


## 7. paso: crear el archivo setup_nfs_Server.yml
```yaml
---
- name: Configuración del servidor NFS
  hosts: nfs
  become: yes
  
  vars_files:
    - ../vars/variables.yml

  tasks:

    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    - name: Actualizar los paquetes
      apt:
        upgrade: dist
        update_cache: yes

    - name: Instalar el servidor NFS
      apt:
        name: nfs-kernel-server
        state: present

    - name: Crear el directorio /var/www/html
      file:
        path: /var/www/html
        state: directory
        owner: nobody
        group: nogroup
        mode: '0755'

    - name: Configurar los directorios exportados en /etc/exports
      lineinfile:
        path: /etc/exports
        line: "/var/www/html {{ CLIENT_IP }}(rw,sync,no_subtree_check,no_root_squash)"
        create: yes

    - name: Reiniciar el servicio NFS
      service:
        name: nfs-kernel-server
        state: restarted
```

## 8. paso: crear el archivo setup_nfs_Client.yml
```yaml
- name: Configurar Cliente NFS
  hosts: frontend1
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:

    - name: Actualizar /etc/fstab para montar el directorio NFS al iniciar
      lineinfile:
        path: /etc/fstab
        line: "{{ SERVER_IP }}:/var/www/html /var/www/html nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0"
        create: yes

    - name: Instalar cliente NFS
      apt:
        name: nfs-common
        state: present

    - name: Montar directorio NFS
      mount:
        path: "{{ WORDPRESS_DIRECTORY }}"
        src: "{{ SERVER_IP }}:{{ WORDPRESS_DIRECTORY }}"
        fstype: nfs
        opts: "auto,nofail,noatime,nolock,intr,tcp,actimeo=1800"
        state: mounted

    - name: Actualizar /etc/fstab para montar el directorio NFS al iniciar
      lineinfile:
        path: /etc/fstab
        line: "{{ SERVER_IP }}:/var/www/html /var/www/html nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0"
        create: yes
```

## 9. paso: crear el archivo deploy_wordpress_backend.yml
```yaml
---
- name: Configurar Base de Datos para WordPress
  hosts: backend
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:
    # Instalar python3-pymysql usando apt
    - name: Asegurarse de que python3-pymysql esté instalado
      apt:
        name: python3-pymysql
        state: present
        update_cache: yes
      become: yes

    # Instalar python3-pip usando apt
    - name: Asegurarse de que pip3 esté instalado
      apt:
        name: python3-pip
        state: present
        update_cache: yes
      become: yes

    # Configurar MySQL para aceptar conexiones remotas
    - name: Permitir conexiones remotas en MySQL
      lineinfile:
        path: /etc/mysql/mysql.conf.d/mysqld.cnf
        regexp: '^#bind-address.*'
        line: 'bind-address = 0.0.0.0'
      become: yes

    - name: Reiniciar servicio MySQL
      systemd:
        name: mysql
        state: restarted
      become: yes

    # Crear la base de datos de WordPress
    - name: Crear la base de datos de WordPress
      mysql_db:
        name: "{{ WORDPRESS_DB_NAME }}"   # Asegúrate de que sea WORDPRESS_DB_NAME
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
      become: yes

    # Crear usuario de WordPress y asignar privilegios
    - name: Crear usuario de WordPress
      mysql_user:
        name: "{{ WORDPRESS_DB_USER }}"
        password: "{{ WORDPRESS_DB_PASSWORD }}"
        host: "{{ FRONTEND_PRIVATE_IP }}"
        priv: "{{ WORDPRESS_DB_NAME }}.*:ALL"
        state: present
        login_unix_socket: /var/run/mysqld/mysqld.sock 
      become: yes


```

## 10. paso: crear el archivo deploy_wordpress_frontend.yml
```yaml
---
- name: Desplegar Frontend de WordPress
  hosts: frontend1
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:

    # Eliminamos el directorio de instalación si existe
    - name: Eliminar el directorio de instalación
      file:
        path: /usr/local/bin/wp/
        state: absent

    # Configuramos los permisos en el directorio donde voy a instalar de WP-CLI
    - name: Configurar permisos de wp-cli
      file:
        path: /usr/local/bin/
        owner: www-data
        group: www-data
        mode: 0755

    # Descargamos WP-CLI para gestionar WordPress desde la terminal
    - name: Descargar la utilidad WP-CLI
      get_url:
        url: https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
        dest: /usr/local/bin/wp
        mode: 0755

    # Eliminamos instalaciones previas de Wordpress en /var/www/html
    - name: Eliminar instalaciones previas de WordPress
      shell: rm -rf /var/www/html/*
      
    # Modificamos los permisos del directorio de instalación de Wordpress
    - name: Modificar permisos del directorio /var/www/html
      file:
        path: /var/www/html
        owner: www-data
        group: www-data
        recurse: yes
        mode: 0755

    # Descargamos el código fuente de WordPress usando WP-CLI
    - name: Descargar el código fuente de WordPress
      command: 
        wp core download \
        --locale="{{ wordpress_locale }}" \
        --path=/var/www/html \
        --allow-root

    # Eliminamos el archivo wp-config.php en caso de que exista
    - name: Eliminar wp-config.php si existe
      file:
        path: /var/www/html/wp-config.php
        state: absent

    # Creamos el archivo de configuración de WordPress con los datos del archivo de variables
    - name: Crear el archivo de configuración de WordPress
      command: 
        wp config create \
        --dbname="{{ WORDPRESS_DB_NAME }}" \
        --dbuser="{{ WORDPRESS_DB_USER }}" \
        --dbpass="{{ WORDPRESS_DB_PASSWORD }}" \
        --dbhost="{{ WORDPRESS_DB_HOST }}" \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Instalamos WordPress con los datos del archivo de variables
    - name: Instalar WordPress
      command: 
        wp core install \
        --url="{{ LE_DOMAIN }}" \
        --title="{{ wordpress_title }}" \
        --admin_user="{{ wordpress_admin_user }}" \
        --admin_password="{{ wordpress_admin_password }}" \
        --admin_email="{{ wordpress_admin_email }}" \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Cambiamos los permisos del directorio /var/www/html
    - name: Configuramos permisos de /var/www/html
      file:
        path: /var/www/html
        owner: www-data
        group: www-data
        mode: '0755'
        recurse: yes

    # Cambiamos los permisos del directorio /wp-content/upgrade
    - name: Configuramos permisos de wp-content
      file:
        path: /var/www/html/wp-content/upgrade
        owner: www-data
        group: www-data
        mode: '0755'
        recurse: yes

    # Cambiamos los permisos del directorio /wp-content/plugins
    - name: Configuramos permisos de wp-content
      file:
        path: /var/www/html/wp-content/plugins
        owner: www-data
        group: www-data
        mode: '0755'
        recurse: yes

    # Instalamos y activamos el tema de Mindscape para el WordPress
    - name: Instalar y activar el tema Mindscape
      command: 
        wp theme install mindscape --activate \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Instalamos y activamos el plugin wps-hide-login para proteger el acceso al panel de administración
    - name: Instalar y activar el plugin wps-hide-login
      command: 
        wp plugin install wps-hide-login --activate \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Configuramos la URL de acceso oculta mediante wps-hide-login con los datos del archivo de variables
    - name: Configurar el plugin wps-hide-login
      command: 
        wp option update whl_page {{ wordpress_hide_login_url }} \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Configuramos los enlaces permanentes para la estructura de URLs
    - name: Configurar los enlaces permanentes
      command: 
        wp rewrite structure '/%postname%/' \
        --path="{{ WORDPRESS_DIRECTORY }}" \
        --allow-root

    # Copiamos el archivo .htaccess al directorio de WordPress
    - name: Copiar el archivo .htaccess al directorio de WordPress
      copy:
        src: ../htaccess/.htaccess
        dest: "{{ WORDPRESS_DIRECTORY }}/.htaccess"
        owner: www-data
        group: www-data
        mode: '0644'

    # Nos aseguramos de que la variable $_SERVER['HTTPS'] esté en el archivo wp-config.php
    - name: Asegurar que $_SERVER['HTTPS'] esté en wp-config.php
      lineinfile:
        path: /var/www/html/wp-config.php
        line: "$_SERVER['HTTPS'] = 'on';"
        state: present
        insertbefore: "^define\\( 'DB_NAME', 'wordpress' \\);"

    # Reiniciamos Apache para aplicar los cambios
    - name: Reiniciar el servidor web Apache
      service:
        name: apache2
        state: restarted
```

## 11. paso: crear el archivo setup_loadbalancer.yml
```yaml
- name: Configurar Nginx como balanceador de carga
  hosts: load_balancer
  become: yes

  vars_files:
    - ../vars/variables.yml

  tasks:

    # Actualizamos los repositorios para tener las últimas versiones de los paquetes
    - name: Actualizar los repositorios
      apt:
        update_cache: yes

    # Actualizamos los paquetes del sistema, con 'dist' actualizamos los paquetes del sistema.
    - name: Actualizar los paquetes
      apt:
        upgrade: dist  

    # Instalamos Nginx 
    - name: Instalar Nginx
      apt:
        name: nginx
        state: present

    # Copiamos la configuración personalizada del balanceador de carga desde mi template.
    - name: Copiar configuración de balanceador de carga
      template:
        src: ../templates/load_balancer.conf
        dest: /etc/nginx/sites-available/load_balancer.conf

    # Deshabilitamos la configuración por defecto de Nginx.
    - name: Deshabilitar el sitio por defecto de Nginx
      file:
        path: /etc/nginx/sites-enabled/default  # (ubicación del archivo de conf)
        state: absent  # Lo elimino

    # Sustituimos IP_FRONTEND_1 en la configuración con la IP real desde las variables.
    - name: Sustituir IP_FRONTEND_1 en la configuración
      replace:
        path: /etc/nginx/sites-available/load_balancer.conf
        regexp: 'IP_FRONTEND_1'  
        replace: '{{ IP_FRONTEND_1 }}'

    # Lo mismo para la IP_FRONTEND_2.
    - name: Sustituir IP_FRONTEND_2 en la configuración
      replace:
        path: /etc/nginx/sites-available/load_balancer.conf
        regexp: 'IP_FRONTEND_2'
        replace: '{{ IP_FRONTEND_2 }}'

    # Sustituimos el dominio de Let's Encrypt en la configuración del balanceador.
    - name: Sustituir LE_DOMAIN en la configuración
      replace:
        path: /etc/nginx/sites-available/load_balancer.conf
        regexp: 'LE_DOMAIN'  # Buscamos el marcador de dominio.
        replace: '{{ LE_DOMAIN }}'  # Lo reemplazo con el valor real.

    # Creamos un enlace simbólico para habilitar la configuración del balanceador de carga en Nginx.
    - name: Habilitar la configuración de Nginx
      file:
        src: /etc/nginx/sites-available/load_balancer.conf
        dest: /etc/nginx/sites-enabled/load_balancer.conf
        state: link

    # Deshabilitamos la configuración 000-default en Nginx.
    - name: Deshabilitar 000-default en Nginx
      file:
        path: /etc/nginx/sites-available/000-default.conf
        state: absent 

    # Finalmente, reiniciamos Nginx para que aplique los cambios.
    - name: Reiniciar Nginx
      systemd:
        name: nginx
        state: reloaded  # uso reload para recargar el servicio sin necesidad de reiniciarlo
```

## 12. paso: crear el archivo setup_letsencrypt_certificate.yml
```yaml
---
- name: Instalar y Configurar Let's Encrypt con Certbot
  hosts: load_balancer
  become: yes
  vars_files:
    - ../vars/variables.yml
  tasks:

    # Actualizamos los repositorios para tener las últimas versiones de los paquetes
    - name: Actualizar la lista de paquetes
      apt:
        update_cache: yes

    # Eliminamos cualquier versión previa de Certbot 
    - name: Desinstalar instalaciones previas de Certbot en caso de que existan
      apt:
        name: certbot
        state: absent

    # Instalamos Certbot con snap
    - name: Instalar Certbot con Snap
      snap:
        name: certbot
        classic: yes
        state: present

    # Solicitamos y configuramos automáticamente el certificado SSL mi dominio
    - name: Solicitar y configurar certificado con Cerbot con Nginx
      command:
        cmd: certbot --nginx -m {{ LE_EMAIL }} --agree-tos --no-eff-email -d {{ LE_DOMAIN }} 

```
## una vez hayamos creado todos los archivos hay que crear 2 archivos maas que serán para los archivos de configuración, habrá que crear 2 que son los siguientes:
### 000-default.conf
```yaml

<VirtualHost *:80>
    ServerName practica-iaw.zapto.org
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html/

    DirectoryIndex index.php index.html

    <Directory /var/www/html>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### load_balancer.conf
```yaml
upstream frontend_servers {
    server IP_FRONTEND_1;
    server IP_FRONTEND_2;
}
server {
    listen 80;
    server_name practica-iaw.zapto.org;

    location / {
        proxy_pass http://frontend_servers;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    
    }
}
```
## ahora tendremos que crear un archivo exports que será para configurar los directorios compartidos
```yaml
/var/www/html  CLIENT_IP(rw,sync,no_root_squash,no_subtree_check)
```

## y también crear un archivo .htacces para las reescrituras de las urls
```yaml
# BEGIN WordPress
<IfModule mod_rewrite.c>
RewriteEngine On
RewriteBase /
RewriteRule ^index\.php$ - [L]
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule . /index.php [L]
</IfModule>
# END WordPresss
```

## una vez hecho todo esto, ya tendriamos todo creado y habría que ejecutarlo en el siguiente orden:
```yaml
---
- import_playbook: playbooks/install_lamp_backend.yml
- import_playbook: playbooks/install_lamp_frontend.yml
- import_playbook: playbooks/setup_nfs_server.yml
- import_playbook: playbooks/setup_nfs_client.yml
- import_playbook: playbooks/deploy_wordpress_backend.yml
- import_playbook: playbooks/deploy_wordpress_frontend.yml
- import_playbook: playbooks/setup_load_balancer.yml
- import_playbook: playbooks/setup_letsencrypt_certificate.yml
```

## cuando lo hayamos ejecutado, vamos a comprobar que todo ha funcionado perfectamente, para ello nos vamos a ir al dominio y podemos ver que se hizo perfectamente el wordpress con su balanceo de carga y también podemos ver el certificado de letsencrypt:

![](imagenes/wordpress-inicio.PNG)

![](imagenes/nada.png)


![](imagenes/certificado.png)
