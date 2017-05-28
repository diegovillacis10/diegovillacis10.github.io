---
layout: post
title:  "Probando playbooks de Ansible con Vagrant"
date:   2017-02-26 12:05:00
categories: infrastructure
---

# Probando playbooks de Ansible con Vagrant

En el internet puedes encontrar varios ejemplos de [cómo probar tus playbooks de Ansible usando Vagrant](https://docs.ansible.com/ansible/guide_vagrant.html){:target="_blank"}. Pero, eso nos ayuda
al momento de querer aprovisionar la máquina de Vagrant, colocando en el Vagrantfile algo como esto:

```ruby
Vagrant.configure(2) do |config|
  config.vm.box = "ubuntu/trusty64"
  config.vm.provision "ansible" do |ansible|
    ansible.verbose = "v"
    ansible.playbook = "playbook.yml"
  end
end
```

Con esto, al momento de iniciar la máquina de Vagrant `vagrant up`, y usando el comando de
aprovisionamiento `vagrant provision` el playbook de ansible `playbook.yml` (que existe al mismo
nivel que el `Vagrantfile`) será llamado por vagrant para que aprovisione la máquina.

### ¿Pero qué pasa cuando queremos usar el comando `ansible-playbook` para ejecutar playbooks en una
máquina de vagrant ya levantada?

Para este ejemplo usaremos una máquina de ubuntu de vagrant para virtualbox, corriendo en la
terminal el siguiente comando, el cual ya nos generará el `Vagrantfile`.

```sh
$ vagrant init ubuntu/trusty64; vagrant up --provider virtualbox
```

Sabemos que podemos usar el comando `vagrant ssh` para entrar a la máquina virtual el cual usa su
propio sistema de llaves para conectarse, pero si quisiéramos usar el comando:

```sh
$ ssh vagrant@127.0.0.1
```

Obtendremos un mensaje de error cómo este:

![Connection Refused](/assets/ansible_vagrant/connection_refused.png)

Si revisamos un poco cuando creamos la máquina virtual podemos ver la redirección de puertos que
vagrant hace por nosotros:

![Vagrant init](/assets/ansible_vagrant/vagrant_init.png)

Podemos decirle a nuestro comando de ssh que queremos conectarnos a ese puerto.

```sh
$ ssh vagrant@127.0.0.1 -p 2222
```

El problema con este comando es que, logrará hacer la conexión, pero siempre nos pedirá la
contraseña del usuario con el que queremos conectarnos (vagrant) pero eso no es lo que buscamos ya
que si recordamos cómo funciona Ansible, éste se conecta mediante llaves a las máquinas por lo que
tenemos que buscar otra opción.

Y en la mayoría de los casos, usamos `become: yes` para decirle a ansible que ejecute los comandos
como `sudo`.

Por ejemplo, aquí tenemos un pequeño playbook que lo que hace es simplemente instalar git con su
respectivo inventory.

> NOTA: Cabe recalcar que tenemos que colocar la variable `ansible_port=2222` ya que por defecto,
ésta tiene como valor `22`.

`playbook.yml`

```yml
---
- hosts: local
  remote_user: vagrant
  become: yes

  tasks:
  - name: Install git
    apt:
      name: git
      state: installed

```

`inventory`

```
[local]
127.0.0.1 ansible_port=2222
```

Y cuando corremos el playbook obtendremos un mensaje así:

![error playbook](/assets/ansible_vagrant/error_playbook.png)

### ¿Y bueno, cómo arreglamos esto?

Para eso, tenemos que dejarle sabe a la máquina de Vagrant nuestra llave pública. Por lo general
nuestra llave pública se encuentra en `~/.ssh/id_rsa.pub`

> NOTA: Si no tienes llaves creadas, puedes seguir la guía de GitHub para [generar una nueva llave SSH](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/){:target="_blank"}.

Para copiar nuestra llave podemos usar el siguiente comando (Sólo para OSX):

```sh
$ pbcopy < ~/.ssh/id_rsa.pub
```

Con esto tenemos copiada nuestra llave pública en el portapapeles.
A continuación lo que tenemos que hacer es agregarla a la lista de las llaves que están autorizadas
para conectarse mediante ssh a la máquina virtual. Este archivo se llama `authorized_keys` y lo
podemos encontrar en `~/.ssh/authorized_keys`.

Para esto vamos a entrar a la máquina virtual de vagrant y pegar nuestra llave pública que la
copiamos anteriormente.

```sh
$ vagrant ssh
$ sudo vi ~/.ssh/authorized_keys
```
Con eso abriremos el editor de texto VIM (se puede usar nano o cualquier otro de preferencia,
recordar que el editor a escoger tiene que estar instalado en la máquina virtual) y pegaremos
nuestra llave justo después de la llave de vagrant (podemos ver que dice "vagrant" al final) y
grabamos el archivo.

Con eso hecho ya podemos salir de la máquina y volvemos a intentar el comando:

```sh
$ ssh vagrant@127.0.0.1 -p 2222
```
Y veremos que ahora ya no tenemos el mensaje de "Connection refused" y ahora vemos otro mensaje:

![Add fingerprint](/assets/ansible_vagrant/add_fingerprint.png)

Cuando escribamos la palabra `yes` observaremos un mensaje que nos dice que ese host se ha agregado
permanentemente a nuestra lista de `known_hosts`. Podemos observar ese cambio ejecutando el
siguiente comando:

```sh
$ cat ~/.ssh/known_hosts
```

Podemos ver que al final del archivo existe una línea que comienza con `[127.0.0.1]:2222` que es el
host que hemos agregado.

Finalmente si volvemos a probar el comando para conectarnos mediante ssh no deberíamos tener
problemas para hacerlo.

Con esto ya podemos ejecutar nuestros playbooks de ansible localmente en la máquina de vagrant 😃

![playbook running](/assets/ansible_vagrant/playbook_running.png)

🎉
