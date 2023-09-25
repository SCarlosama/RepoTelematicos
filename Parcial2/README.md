# Parcial 2 Servicios Telematicos

- Nicolas Escandon
- Santiago Carlosama



## Desarrollo
#### Descativar selinux en todas las maquinas


```
nano /etc/selinux/config
```
se debe instalar vim o nano en todas las maquinas


```
SELINUX=disabled
```
---
### Maquina Firewall
  
#### Detenemos los servicios de NetworkManager para evitar conflictos

```
sudo service NetworkManager stop
sudo chkconfig NetworkManager off
```
#### Iniciar el Firewall


```
sudo service firewalld start
```
#### Configurar Firewall:

#### Remover cualquier interfaz que no sea la correcta (dejar limpia)
```
firewall-cmd --zone=internal --remove-interface=eth0 --permanent 
firewall-cmd --zone=internal --remove-interface=eth1 --permanent
firewall-cmd --zone=internal --remove-interface=eth2 --permanent
```
#### Añadir el eth2 de la zona interna (ip privada) 

```
firewall-cmd --zone=internal --add-interface=eth2 --permanent
firewall-cmd --zone=internal --add-masquerade --permanent
```
#### Remover cualquier interfaz que no sea la correcta (dejar limpia)
```
firewall-cmd --zone=public --remove-interface=eth0 --permanent 
firewall-cmd --zone=public --remove-interface=eth2 --permanent
firewall-cmd --zone=public --remove-interface=eth1 --permanent
```
#### Añadir el eth1 de la zona publica (ip publica)
```
firewall-cmd --zone=public --add-interface=eth1 --permanent
firewall-cmd --zone=public --add-masquerade --permanent
```
#### Añadimos los servicios de la maquina esclavo 
```
firewall-cmd --zone=public --add-service=ftp --permanent
firewall-cmd --zone=public --add-service=dns --permanent
firewall-cmd --zone=public --add-port=53/tcp --permanent
firewall-cmd --zone=public --add-port=53/udp --permanent
```
#### Redireccion de los puertos para la conexion segura de FPT
```
firewall-cmd --zone=public --add-forward-port=port=10100:proto=tcp:toport=22:toaddr=192.168.100.2 --permanent
```
#### Redirecciones de tcp/udp a la máquina DNS esclava
```
firewall-cmd --zone=public --add-forward-port=port=53:proto=tcp:toport=53:toaddr=192.168.100.2 --permanent
firewall-cmd --zone=public --add-forward-port=port=53:proto=udp:toport=53:toaddr=192.168.100.2 --permanent
```

#### Reiniciamos el servicio
```
sudo firewall-cmd --reload
```
#### Verificación de las zonas
```
sudo firewall-cmd --list-all-zones
```
#### configurar el resolv del firewall

```
nameserver 192.168.100.2
```
---
### Maquina Esclavo
#### Instalar el servicio named (instalar tambien vim o nano)
```
sudo yum install bind-utils bind-libs bind-* -y
```
#### Configurar Zonas
ir a `/etc/named.conf` y colocar la siguiente configuracion donde `192.168.100.2` es la ip de la maquina esclavo
```
options {
        listen-on port 53 { 127.0.0.1; 192.168.100.2; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { localhost; 192.168.100.0/24; };
        
        .
        .
        .
```

```
zone "pochita.com" IN {
	type slave;
	file "slaves/pochita.com.fwd";
	masters{ 192.168.100.4; };
};

zone "100.168.192.in-addr.arpa" IN {
	type slave;
	file "slaves/pochita.com.rev";
	masters{ 192.168.100.4; };
};
```
#### Iniciar el servicio named
```
service named start
```
#### Configurar ssl (opcional?)
```
#Podemos verificar primero si ya está instalado el openssl
#rpm -q openssl
yum install openssl
```
### Generar certidicado
```
openssl genrsa -out ca.key 2048

openssl req -new -key ca.key -out ca.csr -subj "/C=CO/ST=Valle del cauca/L=Cali/O=UAO/OU=Informatica/CN=Esclavo/emailAddress=nicolas.escandon@uao.edu.co"

openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt -days 365

cp ca.crt /etc/pki/tls/certs/

cp ca.key /etc/pki/tls/private/

cp ca.csr /etc/pki/tls/private/

chmod 600 /etc/pki/tls/certs/ca.crt
chmod 600 /etc/pki/tls/private/ca.key
chmod 600 /etc/pki/tls/private/ca.csr

yum install mod_ssl -y
```
#### Instalar el servicio vsftpd
```
sudo yum install vsftpd -y
```
ir a `/etc/vsftpd/vsftpd.conf` y reemplazar con la siguiente configuracion:
```
listen_port=10100
anonymous_enable=YES
local_enable=YES
write_enable=YES
local_umask=022
dirmessage_enable=YES
xferlog_enable=YES
connect_from_port_20=YES
xferlog_std_format=YES
ascii_upload_enable=YES
ascii_download_enable=YES
listen=NO
listen_ipv6=YES
ftpd_banner=Bienvenidos al servicio seguro FTP de Pochita.
pam_service_name=vsftpd
userlist_enable=YES
tcp_wrappers=YES
ssl_enable=YES
allow_anon_ssl=YES
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1=YES
ssl_sslv2=YES
ssl_sslv3=YES
require_ssl_reuse=YES
ssl_ciphers=HIGH
rsa_cert_file=/etc/pki/tls/certs/ca.crt
rsa_private_key_file=/etc/pki/tls/private/ca.key
```
#### Iniciar el servicio vsftpd
```
sudo service vsftpd start
```

#### configurar la ip en el  /etc/resolv.conf
```
nameserver 192.168.100.2
```

#### Reiniciamos los servicios para los cambios
```
sudo service named restart
sudo service vsftpd restart
```
---
### Configurar maquina Maestro

#### Instalar nano o vim y desactivar el selinux
#### instalar named
```
sudo yum install bind-utils bind-libs bind-* -y
```
#### Configurar lo siguiente en `/etc/named.conf`
```
options {
        listen-on port 53 { 127.0.0.1; 192.168.100.4; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
		
		forward only;
		forwarders { 192.168.100.4; };
        allow-query     { localhost; 192.168.100.0/24; };
		allow-transfer { 192.168.100.2; };
        
        ...
```
```
zone "pochita.com" IN {
	type master;
	file "pochita.com.fwd";
};

zone "100.168.192.in-addr.arpa" IN {
	type master;
	file "pochita.com.rev";
};
```
#### contenido de `pochita.com.fwd`
```
$ORIGIN pochita.com.
$TTL 3H
@       IN      SOA     maestro.pochita.com. root@pochita.com. (
                        0          ; serial
                        1D         ; refresh
                        1H         ; retry
                        1W         ; expire
                        3H         ; minimum TTL
                        )
@       IN      NS      maestro.pochita.com.
@       IN      NS      esclavo.pochita.com.
@       IN      NS      firewall.pochita.com.
@       IN      NS      firewalloutside.pochita.com.

;host en la zona

@       IN      A       192.168.100.4
@       IN      A       192.168.100.2
@       IN      A       192.168.100.3
@       IN      A       209.191.100.3
maestro         IN      A       192.168.100.4
esclavo         IN      A       192.168.100.2
firewall        IN      A       192.168.100.3
firewalloutside IN      A       209.191.100.3

```
#### contenido de `pochita.com.rev`
```$ORIGIN 100.168.192.in-addr.arpa.
$TTL 3H
@       IN      SOA     maestro.pochita.com. root@pochita.com. (
                        0          ; serial
                        1D         ; refresh
                        1H         ; retry
                        1W         ; expire
                        3H         ; minimum TTL
                        )
@       IN      NS      maestro.pochita.com.
@       IN      NS      esclavo.pochita.com.
@       IN      NS      firewall.pochita.com.
@       IN      NS      firewalloutside.pochita.com.

;host en la zona

4       IN      PTR     maestro.pochita.com.
2       IN      PTR     esclavo.pochita.com.
3       IN      PTR     firewall.pochita.com.
3       IN      PTR     firewalloutside.pochita.com.
```
#### Otorgarle los permisos a las zonas
```
sudo chmod 755 /var/named/pochita.com.fwd
sudo chmod 755 /var/named/pochita.com.rev
```

####  Iniciar el servicio named
```
service named start
```

#### "Verificar la correcta construcción de las zonas
```
named-checkzone gnitech.com /var/named/gnitech.com.fwd
named-checkzone 100.168.192.in-addr.arpa /var/named/gnitech.com.rev
```

#### Configuración del resolv.conf `/etc/resolv.conf`
```
nameserver 192.168.100.4
```

#### Reiniciamos el servicio named para aplicar cambios
```
sudo service named restart
```

---
### Pruebas

desde la maquina firewall podemos ejecutar los siguientes comandos:
#### Realizar ping de verificacion
```
ping -c 5 maestro.pochita.com
ping -c 5 esclavo.pochita.com
ping -c 5 firewall.pochita.com
ping -c 5 firewalloutside.pochita.com
```
<p aling="center">
    <img src="readmeImages\pingTest.JPG"/>     
</p>

#### Verificacion por nslookup
```
nslookup esclavo.pochita.com
nslookup maestro.pochita.com
```

<p aling="center">
    <img src="readmeImages\nslookupTest.JPG"/>     
</p>