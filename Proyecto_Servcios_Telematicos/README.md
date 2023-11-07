
# Proceso de configuración Proyecto 1 HAProxy - Pruebas con Artillery 

1. Primeros pasos creación de los servidores, 

***************************************************************

configuración de Vagrantfile


Vagrant.configure("2") do |config|
  if Vagrant.has_plugin?("vagrant-vbguest")
    config.vbguest.no_install = true
    config.vbguest.auto_update = false
    config.vbguest.no_remote = true
  end

  config.vm.define :haproxy do |haproxy|
    haproxy.vm.box = "fepe/stream8"
    haproxy.vm.network :private_network, ip: "10.85.32.191"
    haproxy.vm.hostname = "haproxy"
    haproxy.vm.synced_folder ".", "/vagrant"
  end

  config.vm.define :sender do |sender|
    sender.vm.box = "fepe/stream8"
    sender.vm.network :private_network, ip: "10.85.32.172"
    sender.vm.hostname = "sender"
    sender.vm.synced_folder ".", "/vagrant"
  end

  config.vm.define :engine do |engine|
    engine.vm.box = "fepe/stream8"
    engine.vm.network :private_network, ip: "10.85.32.225"
    engine.vm.hostname = "engine"
    engine.vm.synced_folder ".", "/vagrant"
  end
  
  config.vm.define :inbound do |inbound|
    inbound.vm.box = "fepe/stream8"
    inbound.vm.network :private_network, ip: "10.85.32.46"
    inbound.vm.hostname = "inbound"
    inbound.vm.synced_folder ".", "/vagrant"
  end
end

***************************************************************

Acondicionamiento de los servidores

1. Instalar algunas herramientas para configuración de la red
sudo yum install net-tools

2. Instalar el editor Vim
sudo yum install vim


3. Desactiva SELINUX servicio de seguridad de linux
ruta:  cd /etc/selinux
sudo vim config
se pone en estado: disabled

4. luego de hacer este proceso en cada uno de los servidores proceder a detenerlos y reniciarlos 
con el siguiente comando

a. vagrant halt 
b. vagrant up

Ejemplo: ver el status: sestatus


Validar el selinux desactivado
sestatus

******************************************************

Configuración servidor con HAProxy

validar selinux
con un sestatus

instalación haproxy
sudo yum install haproxy


configurcion de haproxy
cd /etc/haproxy/

ingresar a la configuracion haproxy
sudo vim haproxy.cfg

se adiciona la siguiente configuración en haproxy.cfg

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

    # utilize system-wide crypto-policies
    ssl-default-bind-ciphers PROFILE=SYSTEM
    ssl-default-server-ciphers PROFILE=SYSTEM

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000
	
	
#---------------------------------------------------------------------
# frontend stats
#---------------------------------------------------------------------

frontend servidores_web
    bind 10.85.32.191:4685
    mode http
    log global

    maxconn 10

    timeout client 100s

    stats enable
    stats hide-version
    stats refresh 30s
    stats show-node
    stats auth fepe:fepe*15!
    stats uri /haproxy?estadistica

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend main
    bind *:80
    # acl url_static       path_beg       -i /static /images /javascript /stylesheets
    # acl url_static       path_end       -i .jpg .gif .png .css .js

    # use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app

    #----------------------------------------------------#
    # Algoritmos de balanceo -- HAProxy                  #
    #----------------------------------------------------#
    # balance     roundrobin
    # balance     leastconn
    balance     source
    timeout server 100s
    timeout connect 100s
    timeout queue 100s
    server  sender 10.85.32.172:80 check
    server  engine 10.85.32.225:80 check
    server  inbound 10.85.32.46:80 check
	
************************************************************************

Ejecutar los siguientes comandos para iniciar el servicio de HAProxy

sudo systemctl status haproxy.service
sudo systemctl start haproxy.service
sudo systemctl stop haproxy.service
sudo systemctl enable haproxy

validar que el sitio web de HAProxy

10.85.32.191:4685/haproxy?estadistica
usuario: fepe
password: fepe*15!


Validar el selinux desactivado
sestatus

validar si el firewalld esta arriba dado el caso detenerlo
sudo service firewalld status
sudo service firewalld stop


**********************************************************

Se ajusta el servidor Sender para resolver nombres de dominio 

cd /etc/

sudo vim named.conf


options {
        listen-on port 53 { 127.0.0.1; 10.85.32.172;};
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; 10.85.32.0/24;};

        recursion yes;

        dnssec-enable yes;
        dnssec-validation yes;

        managed-keys-directory "/var/named/dynamic";

        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";

        /* https://fedoraproject.org/wiki/Changes/CryptoPolicy */
        include "/etc/crypto-policies/back-ends/bind.config";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
        type hint;
        file "named.ca";
};

/* Servidor Sender */
/* Zona hacia adelante */
zone "sender.com" IN {
        type master;
        file "sender.com.fwd";
};

/* Zona reversa */
zone "32.85.10.in-addr.arpa" IN {
        type master;
        file "reversa.com.rev";
};


/* Servidor Engine */
/* Zona hacia adelante */
zone "engine.com" IN {
        type master;
        file "engine.com.fwd";
};

/* Servidor Inbound */
/* Zona hacia adelante */
zone "inbound.com" IN {
        type master;
        file "inbound.com.fwd";
};

/* Servidor HAProxy */
/* Zona hacia adelante */
zone "haproxy.com" IN {
        type master;
        file "haproxy.com.fwd";
};
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
                                  
								  
********************************************************

crear los archivos de zonas en la ruta con super usuario debido a que se 
requiere permisos de administrador

cd /var/named/

cp named.empty sender.com.fwd
cp named.empty engine.com.fwd
cp named.empty inbound.com.fwd
cp named.empty reversa.com.rev



engine.com.fwd *****************************************

$ORIGIN engine.com.
$TTL 3H
@       IN SOA  server.engine.com. root@engine.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

@       IN      NS     server.engine.com.

server  IN      A      10.85.32.225
ww2     IN      CNAME  server



sender.com.fwd *****************************************

$ORIGIN sender.com.
$TTL 3H
@       IN SOA  server.sender.com. root@sender.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

@       IN      NS     server.sender.com.

server  IN      A      10.85.32.172
ww2     IN      CNAME  server



inbound.com.fwd *****************************************

$ORIGIN inbound.com.
$TTL 3H
@       IN SOA  server.inbound.com. root@inbound.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

@       IN      NS     server.inbound.com.

server  IN      A      10.85.32.46
ww2     IN      CNAME  server





reversa.com.rev *****************************************

$ORIGIN 32.85.10.in-addr.arpa.
$TTL 3H
@       IN SOA  server.sender.com. root@sender.com. (
                                        0       ; serial
                                        1D      ; refresh
                                        1H      ; retry
                                        1W      ; expire
                                        3H )    ; minimum

@       IN      NS     server.sender.com.

172     IN      PTR    server.sender.com.     ; Para 10.85.32.172
225     IN      PTR    server.engine.com.     ; Para 10.85.32.225
46      IN      PTR    server.inbound.com.    ; Para 10.85.32.46
191     IN      PTR    server.haproxy.com.    ; Para 10.85.32.191
~                                                                  



chmod 755 sender.com.fwd 
chmod 755 engine.com.fwd
chmod 755 inbound.com.fwd
chmod 755 reversa.com.rev


validar los archivos de zona que esten creados correctamente

named-checkzone sender.com /var/named/sender.com.fwd 
named-checkzone engine.com /var/named/engine.com.fwd 
named-checkzone inbound.com /var/named/inbound.com.fwd


verificar zona reversa
named-checkzone 32.85.10.in-addr.arpa /var/named/reversa.com.rev

Administración del servicio iniciar el named

sudo systemctl status named.service
sudo systemctl start named.service
sudo systemctl stop named.service
sudo systemctl enable named.service

************************************************************
Instalar httpd en los servidores [sender, engine, inbound]


validar que esta instalado httpd

rpm -q httpd

instalar http con el siguiente comando
sudo yum install httpd

cuando se termine la instalación dirigirnos a la siguiente ruta 

ver la ip del servidor

ifconfig
cd /var/www/html/

se crea un index.html 
touch index.html

se edita el archivo html de acuerdo a lo que se requiera
vim index.html

se le da permisos al archivo con el siguiente comando
chmod 755 index.html

se procede a verificar que el servicio este corriendo

sudo systemctl enable httpd 
sudo systemctl status httpd
sudo systemctl start httpd
sudo systemctl stop httpd

el archivo de configuracion se encuentra en cd /etc/httpd/conf
desde el servidor

luego configurar el archivo vim httpd.conf
y se le agrega la siguiente linea 

se ajusta esta configuración 

# Further relax access to the default document root:
<Directory "/var/www/html">
    # Options Indexes FollowSymLinks
    AllowOverride All
    # Require all granted
    Order allow,deny
    Allow from all
</Directory>


se procede a crear los host virtuales

<VirtualHost *:80>
    ServerName ww2.sender.com
    DocumentRoot /var/www/html
</VirtualHost>

<VirtualHost *:80>
    ServerName ww2.engine.com
    DocumentRoot /var/www/html
</VirtualHost>

<VirtualHost *:80>
    ServerName ww2.inbound.com
    DocumentRoot /var/www/html
</VirtualHost>


hacer pruebas desde el navegador que pueda acceder 
ww2.inbound.com
ww2.engine.com
ww2.sender.com

validar con un pin si esta funcionando el dns 
ww2.sender.com tiene que resolver esta dirección 

Dado el caso que no pueda resolver ingresar a 

1. panel de control en windows
2. redes e Internet
3. centro de redes y recursos compartidos
4. cambiar configuración del adaptador. 
5. seleccionar el adaptador de red que tenga 
6. click derecho y propiedades
7. seleccion Habilitar el protocolo de internet versión 4(TCP/IPv4)
8. Propiedades
9. en servidor DNS preferido, colocar el servidor que configuro como dns en este caso es el 10.85.32.172

volver a ejecutar las pruebas



****************************************************************************************
en la maquina windows procedemos a instalar NODE.js para que este permita ejecutar npm

https://nodejs.org/en


Artillery
Forma de instalación
npm install -g artillery artillery-engine-playwright

en el navegador podemos acceder a las estadisticas configuradas en HAProxy 
http://ww2.haproxy.com:4685/haproxy?estadistica

se configura un archivo en formato .json que nos servira para las pruebas


configuración fepe_prueba_ligera.json

{
  "config": {
    "target": "http://ww2.haproxy.com/",
    "phases": [
      {
        "duration": 60,
        "arrivalRate": 10,
        "maxVusers": 100
      }
    ]
  },
  "scenarios": [
    {
      "flow": [
        {
          "get": {
            "url": "/"
          }
        },
        {
          "think": 2
        },
        {
          "get": {
            "url": "/about"
          }
        }
      ]
    }
  ]