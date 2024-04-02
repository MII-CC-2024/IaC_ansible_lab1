# Ansible

Caso de uso: Implementar un Balanceador de carga, no gestionado.

Supongamos que disponemos de tres máquinas virtuales con Amazon Linux 2023:

webserver1: 3.228.178.191
webserver2: 3.232.218.105
loadbalancer: 34.235.19.16


## Instalar Ansible en la máquina de control

```
$ pip install --user ansible

 $ ansible --version
ansible [core 2.15.10]
  config file = /home/ec2-user/environment/lab1_ansible/ansible.cfg
  configured module search path = ['/home/ec2-user/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/ec2-user/.local/lib/python3.9/site-packages/ansible
  ansible collection location = /home/ec2-user/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/ec2-user/.local/bin/ansible
  python version = 3.9.16 (main, Sep  8 2023, 00:00:00) [GCC 11.4.1 20230605 (Red Hat 11.4.1-2)] (/usr/bin/python3)
  jinja version = 3.1.3
  libyaml = True
```

## Crear un fichero de inventario: hosts

https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html

```
# Por defecto /etc/ansible/hosts
# Podemos usar otro con -i <inventario> o indicándolo en ansible.cfg

# ./hosts

[webserver]
3.228.178.191
3.232.218.105


[loadbalancer]
34.235.19.16

```

#### Listar los hosts

```
$ ansible all -i hosts --list-hosts
  hosts (4):
    control
    3.228.178.191
    3.232.218.105
    34.235.19.16
```

### Configuración de Ansible

https://docs.ansible.com/ansible/latest/reference_appendices/config.html

La configuración se busca en el siguiente orden:

- Variable de entorno: ANSIBLE_CONFIG
- fichero ansible.cfg en el directorio actual
- fichero ~/.ansible.cfg (en el directorio home)
- Fichero /etc/ansible/ansible.cfg

Crear el fichero ansible.cfg

```
# ansible.cfg

[defaults]
inventory = ./hosts
remote_user = ec2-user
private_key_file =  ./labsuser.pem
```

```
NOTE: Windows WSL: So we cant have a configuration file which can be written by everyone, so you can set permission as 755 to /home/ansible-user/ansible directory and it should work like below example :

ansible-user@IND004513:~/ansible$ sudo chmod 755 /home/ansible-user/ansible/

ansible-user@IND004513:~/ansible$ ansible --list-hosts all
```

### Asignando nombre (Hostname DNS) a los hosts

```
# ./hosts

[webserver]
webapp1 ansible_host=3.228.178.191
webapp2 ansible_host=3.232.218.105


[loadbalancer]
weblb ansible_host=34.235.19.16


```
Es posible utilizar patrones para seleccionar los hosts

```
$ ansible web* --list-hosts
  hosts (3):
    webapp1
    webapp2
    weblb
```

### Recursos

RESOURCES
- [Install the Control Machine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html#installing-the-control-machine)
- [Working With Dynamic Inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_dynamic_inventory.html)
- [Ansible Inventory File](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html)
- [Ansible Configuration Settings](https://docs.ansible.com/ansible/latest/reference_appendices/config.html#ansible-configuration-settings)
- [Working with Patterns](https://docs.ansible.com/ansible/latest/user_guide/intro_patterns.html)


## Comandos Ad-hoc en Ansible

Los comandos Ad-hoc, en Ansible, son la forma de enviar acciones las máquinas del inventario. La sintaxis:

```
$ ansible <options> <host-pattern>
```

```
$ ansible -m ping all
    ^      ^   ^   ^
    '      '   '   '
  command  '   '   '
         flag  '   '
        módulo '   '
               '   '
           nombre  '
           módulo  '
               Inventario 
```
```
$ ansible -a "uname" all
    ^      ^   ^      ^
    '      '   '      '
  ansible  '   '      '
         flag  '      '
       command '      '
               '      '
            comando   '
                      '
                  Inventario 
```
Ejemplos

```
# Ejemplo ejecución de un comando correcto (tb se puede usar -m shell -a "uname" all)
$ ansible -a "uname" all
control | SUCCESS | rc=0 >>
Linux

webapp2 | SUCCESS | rc=0 >>
Linux

weblb | SUCCESS | rc=0 >>
Linux

webapp1 | SUCCESS | rc=0 >>
Linux
```
```
# Ejemplo ejecución de un comando errónea (tb se puede usar -m command -a "false" \!local )
$ ansible -a "false" \!local
webapp1 | FAILED | rc=1 >>
non-zero return code

webapp2 | FAILED | rc=1 >>
non-zero return code

weblb | FAILED | rc=1 >>
non-zero return code
```


### Enlaces de interés

- Comandos ansible: https://docs.ansible.com/ansible/latest/cli/ansible.html
- Índice de módulos de ansible: https://docs.ansible.com/ansible/latest/modules/modules_by_category.html


## Playbooks
https://docs.ansible.com/ansible/latest/user_guide/playbooks_intro.html

Playbooks (libro de jugadas, lista de la compra o receta) es un script que incluye las intrucciones para gestionar la configuración de nuestros sistemas remotos

Permiten  gestionar la configuración y crear infraestructura mediante script de código, utilizando la sintasis Yaml.

Están compuesto por uno o más plays, cuyo objetivo es realizar tareas, mediante un módulo ansible, en los host detallados en el inventario.

Por ejemplo, el comando:

```
$ ansible -m ping all
```
En un playbook sería:

```
---

- name: Chequear conexión a servidores
  hosts: all

  tasks:

  - name: Ping a todos los servidores
    ansible.builtin.ping:
```

Para ejecutar un playbook se utiliza el comando:

```
$ ansible-playbook ping.yml
```

### Tareas a realizar

- Gestión de paquetes: con el módulo correspondiente a la distribución linux instalada
- Configurar infraestructura: copiar (con el módulo copy) y sincronizar ficheros (con el módulo synchronize).
- Gestionar servicios: configurarlo con el módulo lineinfile e iniciarlos, pararlos o restaurarlo con el módulo service

#### Gestión de paquetes

```
# yum-update.yaml

---
- name: Update web a loadbalances
  hosts: webserver:loadbalancer
  become: true
  tasks:
  - name: Updating yum packages
    ansible.builtin.yum: name=* state=latest
```

#### Instalar servicios

```
# install-services.yml

---
- name: Install Apache on loadbalancer
  hosts: loadbalancer
  become: true
  tasks:

  - name: Install the latest version of Apache
    ansible.builtin.yum:
      name: httpd
      state: present
  - name: Ensure apache starts
    service: name=httpd state=started enabled=yes

- name: Install Apache and PHP on WebServers
  hosts: webserver
  become: true
  tasks:
  - name: Installing services
    ansible.builtin.yum:
      name:
      - httpd
      - php
      state: present
  - name: Ensure apache starts
    service: name=httpd state=started enabled=yes
```

#### Instanlación y configuración de la Aplicación

```
# deploy-app.yml

---
- name: Upload App and Confgure PHP
  hosts: webserver
  become: true
  tasks:
  - name: Upload application file
    copy:
      src: app/index.php
      dest: /var/www/html
      mode: 0755

  - name: Configure php.ini file
    lineinfile:
      path: /etc/php.ini
      regexp: ^short_open_tag
      line: 'short_open_tag=On'

  - name: restart apache
    service: name=httpd state=restarted
```

Es posible definir un handler para que apache se restaure si hay algún cambio en la configuración, se mostrará en el siguiente ejemplo.



- Configuración del balanceador

```
# setup-lb.yml
---
- name: Configure loadbalancer
  hosts: loadbalancer
  become: true
  tasks:
  - name: Creating template
    template:
      src: config/lb-config.j2  #config/lb.conf
      dest:  /etc/httpd/conf.d/lb.conf
      owner: bin
      group: wheel
      mode: 064
    notify: restart httpd

  handlers:
  - name: restart httpd
    service:
      name: httpd
      state: restarted
```

El fichero de configuración sería: 

```
ProxyRequests off
<Proxy balancer://webcluster >
    BalancerMember http://3.228.178.191
    BalancerMember http://3.232.218.105
    ProxySet lbmethod=byrequests
</Proxy>

# Optional
<Location /balancer-manager>
  SetHandler balancer-manager
</Location>

ProxyPass /balancer-manager !
ProxyPass / balancer://webcluster/
```

Se podría utilizar Jinja para generar el contenido de forma dinámica (programática)
https://jinja.palletsprojects.com/en/2.11.x/

```
ProxyRequests off
<Proxy balancer://webcluster >
  {% for host in hostvars[inventory_hostname]['groups']['webserver'] %}
    BalancerMember http://{{host}}
  {% endfor %}
    ProxySet lbmethod=byrequests
</Proxy>

# Optional
<Location /balancer-manager>
  SetHandler balancer-manager
</Location>

ProxyPass /balancer-manager !
ProxyPass / balancer://webcluster/
```

### Englobar los playbooks en uno
https://docs.ansible.com/ansible/latest/modules/import_playbook_module.html

```
# all-playbooks.yml
---
  - import_playbook: dnf-update.yml
  - import_playbook: install-services.yml
  - import_playbook: deploy-app.yml
  - import_playbook: setup-lb.yml
```

### Crear un playbook para comprobar el estado de los servicios

```
# check-status.yml
---
- name: Services Status
  hosts: webserver:loadbalancer
  become: true
  tasks:
  - name: Check status of apache
    shell:
      cmd: service httpd status
      warn: False
```

## Uso de Variables 

Las variables permiten crear configuraciones más generales que se comncretan con los valores de estas variables.

Para acceder a las variables se utiliza jinja2.

Ansible dispone de una serie de variables (ansible_facts) que se instancia durante la tarea "Gathering Facts". Puedes consultar estas variable, por ejemplo, para el grupo control mediante:

```
$ ansible -m setup control
```

```yml

- name: Print all available facts
  hosts: loadbalancer
  gather_facts: true

  tasks:

  - name: Show Facts
    ansible.builtin.debug:
      var: ansible_facts
```

En el siguiente ejemplo, se amplia nuestra App con un nuevo fichero info.php incluyendo el nombre del host:

```
# setup-app.yml

---
- hosts: webservers
  become: true
  tasks:
  - name: Upload application file
    copy:
      src: app/index.php
      dest: /var/www/html
      mode: 0755

  - name: Update index.php en la App
    copy: 
      dest: /var/www/html/index.php
      content: "<h1>{{ ansible_hostname }}</h1>"
      
  - name: Configure php.ini file
    lineinfile:
      path: /etc/php.ini
      regexp: ^short_open_tag
      line: 'short_open_tag=On'
    notify: restart apache

  handlers:
  - name: restart apache
    service: name=httpd state=restarted
```


También, podemos crear variables (vars) locales a un Playbook o crear variables en un fichero e importarlo a los Playbooks.

En este primer ejemplo se crea una variable local al Playbook y se utiliza con la sintaxis jinja2:

```
# playbook.yml
---
  vars:
    path_to_app: "/var/www/html"
  tasks:
    - name: incluir info.php a la aplicación
      copy:
        dest: "{{path_to_app}}//info.php"
        content: "<?php phpinfo(); ?>"
```

Es posible definir variables de tipo diccionario:

```
foo:
  field1: one
  field2: two

# Podemos acceder con:
foo['field1']
foo.field1 
```

También, se pueden crear variables registrando (register) valores dependientes de la ejecución de tareas. Con el [módulo debug](https://docs.ansible.com/ansible/latest/modules/debug_module.html) podemos ver las variables y depurar los playbooks. El siguiente código muestra un ejemplo:

```
# playbook.yml
---
  vars:
    path_to_app: "/var/www/html"
  tasks:
    - name: Contenido directorio
      command: ls -la {{path_to_app}}
      register: contenido

    - name: Mostrar el contenido del directorio
      debug:
        msg: "{{contenido}}"
```

## Roles
https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html

Los roles permiten cargar automáticamente ciertos vars_files, tareas y handlers y este agrupamiento por roles permite compartirlos con otros usuarios.

Partiendo de esta estructura:

```
site.yml
webservers.yml
fooservers.yml
roles/
    webservers/
        tasks/
          - main.yml
        vars/
          - main.yml
        handlers/
          - main.yml
    common/
        tasks/
        handlers/
        files/
        templates/
        vars/
        defaults/
        meta/
```

Los roles se usarían con:

```
---
- hosts: webservers
  roles:
    - common
    - webservers
```
El comportamiento para cada rol "x" sería:

- Si existe roles/x/task/main.yml, las tareas incluidas en el main.yml se agregarán al Play.
- Si existe roles/x/handlers/main.yml, los handlers serán agregados al play.
- Si existe roles/x/vars/main.yml, se agregarán las variables.
- Si existe roles/x/defaults/main.yml, se agregarán las variables.
- Si existe roles/x/meta/main.yml, se agregarán las dependencias de roles.
- Cualquier copia, script, plantilla o tareas incluidas (en el rol) pueden hacer referencia a archivos incluidos es roles/x/{archivos, plantillas, tareas}/.

### Comando Ansible Galaxy 

Ansible Galaxy es un sitio gratuito para buscar, descargar, calificar y revisar todo tipo de roles de Ansible desarrollados por la comunidad y puede ser una excelente manera de comenzar sus proyectos de automatización.

El comando ansible-galaxy está incluido en Ansible. El cliente Galaxy le permite descargar roles de Ansible Galaxy, y también proporciona un excelente marco predeterminado para crear sus propios roles.

Podemos crear un rol con:

```
$ ansible-galaxy roles/webservers init
```