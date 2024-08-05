
# TallerJulio2024

El objetivo de este archivo es albergar el contenido y descripción de cómo se usan los playbooks desarrollados.

A su vez, también posee todo el contenido de cada sección y archivo ubicado en el repositorio de GitHub para un más facil acceso a la información.
## Instalación de Ansible

Para la instalación de Ansible, debemos descargar previamente el paquete Python. Los siguientes paquetes se instalarán en el servidor Controller sin utilizar SUDO, excepto Python.

```bash
  # sudo dnf install python3-pip

  // Instala el paquete Pipx para ejecutar aplicaciones de Python.
  $ pip install pipx

  /* Asegura que el directorio de instalación sea en la dirección del sistema,
  con el fin de que se puedan instalar desde cualquier lugar con una consola. */
  $ pipx ensurepath

  // Instala la base de automatización para Ansible.
  $ pipx install ansible-core

  // Inyecta el paquete "argcomplete" que se encarga de brindar autocompletado a los comandos.
  $ pipx inject ansible-core argcomplete

  // Instala el paquete "ansible-lint"
  $ pipx inject ansible-lint

  /* Inyecta el paquete "ansible-lint" que se encarga de verificar el codigo en
     Ansible para que cumpla con unos requisitos minimos. */
  $ pipx inject ansible-core ansible-lint

  // Activa de manera global el autocompletado de argumentos para Python.
  $ activate-global-python-argcomplete --user

  // Carga las definiciones autocompletado para bash, permitiendo el autocompletado en la terminal
  source /home/sysadmin/.bash_completion
```
## Collections

Dicha carpeta, posee un archivo correspondiente a "requirements.yml":

```bash

---
collections:
  - ansible.posix
  - community.general
  - community.mysql

```
## Files

Dicha carpeta posee dos archivos, "tomcat.conf" y "virtualhost.conf", ambos archivos corresponden a la configuración de la aplicación web.

tomcat.conf:

```bash
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
Type=forking
User=sysadmin
Group=sysadmin
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=always
Environment=JAVA_HOME=/usr/lib/jvm/java-11-openjdk

[Install]
WantedBy=multi-user.target


```

virtualhost.conf:

```bash
<Virtualhost *:80>
  ServerName www.todo.com
  ServerAdmin web@todo.com
  DocumentRoot /var/www/todo/html
  <Directory /var/www/todo/html>
  AllowOverride none
  Options Indexes FollowSymLinks
  Require all granted
 </Directory>
</VirtualHost>
```
## Inventory

Dentro de esta carpeta, encontraron dos subcarpetas más y un archivo ".toml" correspondiente a los servidores. A continuación se detallarán los contenidos de dichos elementos:

### group_vars:

centos.yml:

```bash
---
ssh_service: sshd
```

ubuntu.yml:

```bash
---
ssh_service: ssh
```

### host_vars:

server01.yml:

```bash
---
ansible_host: 192.168.56.110
ansible_user: sysadmin
```

server02.yml:

```bash
---
ansible_host: 192.168.56.111
ansible_user: sysadmin
```

### Archivo de servidores:

servidores.toml:

```bash
[centos]
server01        ansible_host=192.168.56.110

[ubuntu]
server02        ansible_host=192.168.56.111

[database]
server02        ansible_host=192.168.56.111  

[linux:children] 
centos
ubuntu
```
## Archivos de configuración de base de datos y servicio web

database.yml:

```bash
---
- name: Configuro servidor base de datos en ubuntu
  hosts: database
  become: true
  user: sysadmin

  tasks:

    - name: UFW Instalado
      ansible.builtin.apt:
        name: ufw
        state: present

    - name: Habilitar puerto 22 en ufw
      community.general.ufw:
        rule: allow
        name: OpenSSH

    - name: Defino políticas de tráfico saliente
      community.general.ufw:
        rule: allow
        direction: out
        state: enabled

    - name: Defino políticas de tráfico entrante
      community.general.ufw:
        rule: allow
        direction: in
        state: enabled

    - name: UFW Servicio UP y Activo
      ansible.builtin.systemd:
        name: ufw
        state: started
        enabled: true

   ### Instalación de Python y sus dependencias
    - name: Instalamos Python 3.x en el servidor Ubuntu
      ansible.builtin.apt:
        name:
          - python3-pip
          - python3-dev
          - python3-pymysql
          - python3-mysql.connector
        state: present

    ### Instalación de base de datos
    
    - name: MariaDB Instalado
      ansible.builtin.apt:
        name: mariadb-server
        state: present
        update_cache: true

    - name: Servidor MariaDB levantado
      ansible.builtin.systemd:
        name: mariadb
        state: started
        enabled: true

    - name: Habilitamos en ufw la conexión a MariaDB
      community.general.ufw:
        rule: allow
        port: "3306"
        protocol: tcp
        direction: in
     

    // Configuración de la base de datos
    - name: Creamos la base de datos todo
      community.mysql.mysql_db:
        check_implicit_admin: true
        login_host: 127.0.0.1
        login_user: root
        login_password: 'tlxroot'
        name: todo
        state: present
        target: ./files/todo.sql

    - name: Configurar virtualhost
      ansible.builtin.copy: 
      src: ./files/virtualhost.conf
      dest: /etc/httpd/conf.d
       

  handlers:
    - name: Restart mariadb
      ansible.builtin.systemd:
        name: mariadb
        state: restarted
```

hardening.yml:

```bash
---
- name: Hardening de servidores
  hosts: linux
  become: true
  user: sysadmin

  tasks:

  - name: Configuro opciones de seguridad de SSH
    ansible.builtin.lineinfile: 
      path: /etc/ssh/sshd_config
      regexp: "^#PermitRootLogin"
      line: PermitRootLogin no
    notify: Reinicio servidor SSH
    register: results_sshd

 
  handlers: 

  - name: Reinicio servidor SSH
    ansible.builtin.systemd_service:
      name: "{{ ssh_service }}"
      state: restarted
```

webserver.yml:

```bash
---

- name: Instalar y configurar un webserver
  hosts: centos
  become: true
  user: sysadmin

  tasks:

  - name: Instalar apache
    ansible.builtin.dnf:
      name: httpd
      state: present

  - name: Configurar virtualhost
    ansible.builtin.copy: 
      src: ./files/virtualhost.conf
      dest: /etc/httpd/conf.d
    notify: Reiniciar apache

  - name: Crear directorio document root
    ansible.builtin.file:
      path: /var/www/todo/html
      state: directory
      mode: '0755'
      owner: apache
      group: apache


  - name: Permito conexiones al puerto 80
    ansible.posix.firewalld:
      service: "{{ item }}"
      state: enabled
      immediate: true
      permanent: true
    loop: 
      - http
      - https

  - name: Apache levantado y habilitado
    ansible.builtin.systemd_service:
      name: httpd
      state: started
      enabled: true

  handlers:

  - name: Reiniciar apache
    ansible.builtin.systemd_service:
      name: httpd
      state: restarted
```

webserver_deploy:

```bash
---
- name: Despliegue de aplicacion ToDo
  hosts: centos
  become: true
  user: sysadmin

  tasks:

    - name: Instalar JDK de java
      ansible.builtin.dnf: 
        name: java-11-openjdk-devel
        state: present

    - name: Instalar Podman
      ansible.builtin.dnf:
        name: podman
        state: present

    - name: Crear grupo Tomcat si no existe
      ansible.builtin.group:
        name: tomcat
        state: present

    - name: Crear usuario Tomcat si no existe
      ansible.builtin.user:
        name: tomcat
        comment: "Tomcat User"
        home: /opt/tomcat
        shell: /bin/nologin
        group: tomcat
        state: present
   
    - name: Asignar permisos correctos al directorio de Tomcat
      ansible.builtin.command:
        cmd: chown -R tomcat:tomcat /opt/tomcat

    - name: Crear directorio para Tomcat
      ansible.builtin.file:
        path: /opt/tomcat
        state: directory
        owner: tomcat
        group: tomcat
        mode: '0755'

    - name: Crear directorio para clonar el repositorio
      ansible.builtin.file:
        path: /tomcat
        state: directory

    - name: Descargar Apache Tomcat
      ansible.builtin.get_url:
        url: https://downloads.apache.org/tomcat/tomcat-9/v9.0.91/bin/apache-tomcat-9.0.91.tar.gz
        dest: /opt/tomcat

    - name: Extraer Apache Tomcat
      ansible.builtin.unarchive:
        src: /opt/tomcat/apache-tomcat-9.0.91.tar.gz
        dest: /opt/tomcat
        remote_src: yes

    - name: Asignar permisos correctos al directorio de Tomcat
      ansible.builtin.command:
        cmd: chown -R tomcat:tomcat /opt/tomcat
    
    - name: Configurar Tomcat como servicio           ### Prompt de chatgpt "configuración tomcat como servicio"
      copy:
        dest: /etc/systemd/system/tomcat.service
        content: |
          [Unit]
          Description=Apache Tomcat Web Application Container
          After=network.target

          [Service]
          Type=forking
          User=tomcat
          Group=tomcat
          Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
          Environment="JAVA_OPTS=-Djava.security.egd=file:///dev/urandom"
          ExecStart=/opt/tomcat/apache-tomcat-9.0.91/bin/startup.sh
          ExecStop=/opt/tomcat/apache-tomcat-9.0.91/bin/shutdown.sh
          SuccessExitStatus=143
          Restart=on-failure

          [Install]
          WantedBy=multi-user.target

    - name: Recargar demonio systemd
      command: systemctl daemon-reload
    
    - name: Habilitar y arrancar servicio Tomcat
      ansible.builtin.systemd:
        name: tomcat
        enabled: yes
        state: started

    - name: Clonar aplicación
      ansible.builtin.git:
        repo: "https://github.com/eoriguela/Tomcat-Todo.git"
        dest: /tomcat

    - name: Desplegar aplicación ToDo
      ansible.builtin.copy:
        src: /tomcat/todo.war
        dest: /opt/tomcat/webapps/todo.war

    - name: Habilitar puertos en el Firewall
      ansible.posix.firewalld:
        port: 8080/tcp
        permanent: yes
        state: enabled

    - name: Recargar Firewall
      ansible.builtin.command:
        cmd: firewall-cmd --reload
```
## A continuación se detallan las capturas de las pruebas de ejecución de los playbooks y funcionamiento de las aplicaciones:

Webserver:

![App Screenshot](https://i.postimg.cc/PJT5ct0J/Webserver.png)

Webserver_deploy.yml:

![App Screenshot](https://i.postimg.cc/8c4zg2VH/Webserver-deploy-yml-Captura-1.png)

Webserver_deploy.yml:

![App Screenshot](https://i.postimg.cc/PrfqQ5mF/Webserver-deploy-yml-Captura-2.png)

Database.yml:

![App Screenshot](https://i.postimg.cc/2j76WZBZ/Database-yml.png)

Hardening.yml:

![App Screenshot](https://i.postimg.cc/rmSpQdn7/Hardening-yml.png)
## Authors

- Ezequiel Origüela - Nº de Estudiante: 280758
- Maximiliano Robles - Nº de Estudiante: 290963

- Repositorio: https://github.com/eoriguela/TallerJulio2024