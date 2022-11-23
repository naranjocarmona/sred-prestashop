# Detalles sobre el entorno

Usaremos una OVA de Ubuntu para configurar VirtualBox para instalar Apache2, PHP 7.4 y Prestashop.
Usaremos otr OVA de Ubuntu para instalar MySQL.

![](/capturas/VERSIONVIRTUALBOX.png)
![](/capturas/VERSIONUBUNTU.png)

# Configuración de la máquina "apache"

Antes de iniciar, nos aseguramos de poner solo una tarjeta de red como adaptador puente.
![](/capturas/CONFIGURACIONAPACHE.png)

# Configuración de la IP estática para el adaptador puente

Modificamos el siguiente fichero para establecer una IP estáticamente:

```
nano /etc/netplan/00-installer-config.yaml
```

En este caso, establecemos la IP 192.168.0.180 con máscara /24, por lo que el contenido del fichero queda así:

![](/capturas/CONFIGURACIONESTATICAAPACHE.png)

Por último aplicamos los cambios:

```
netplan apply
```

![](/capturas/CONFIGURACIONESTATICAAPACHE2.png)

# Instalación de servidor web Apache

Actualizamos los paquetes e instalamos apache:

```
apt update && apt install apache2 -y
```

Vemos que se ha ejecutado:

```
systemctl status apache2
```

![](/capturas/STATUSAPACHE2.png)

Comprobamos en el servidor apache, que desde un navegador, escribiendo la ip de la máquina, que podemos ver la página de bienvenida del servidor apache:

![](/capturas/PAGINAPRINCIPALAPACHE.png)

> Nota: Hay que tener en cuenta que en esta OVA el firewall viene desactivado por lo que todo funciona a la primera. Si el firewall estuviera funcionando, podría bloquear peticiones externas al servidor apache (intentar acceder a la web). Para permitirlo, ejecutariamos el siguiente comando:

```
ufw allow in "Apache"
```

# Instalación PHP

Prestashop requiere una versión específica de PHP (hasta la 7.4), por lo que si solo hicieramos apt install php, instalariamos la ultima versión (8.1), que es incompatible con Prestashop.

Para instalar la versión 7.4 de PHP, ejecutamos los siguientes comandos:

```
apt install software-properties-common -y
add-apt-repository ppa:ondrej/php
apt update
apt install php7.4 -y
apt install libapache2-mod-php7.4 php7.4-mysql -y
```

El último comando instala un par de modulos PHP. Uno que comunica con Apache y otro con MySQL.

# Integración de Apache con PHP.

Apache da preferencia a ficheros HTML frente a PHP. Si no queremos que sea así, tenemos que modificar lo siguiente:

```
nano /etc/apache2/mods-enabled/dir.conf
```

![](/capturas/CONFIGURACION1OPCIONPHP.png)

Después guardamos y reiniciamos el servicio:

```
systemctl reload apache2
```

# Configuración de la máquina "mysql"

Esta máquina solo va a tener una tarjeta de red interna, pero de momento hay que configurarla como adaptador puente porque necesitamos acceso a internet para instalar los paquetes necesarios.

Configuramos VirtualBox como adaptador puente e iniciamos la máquina.


![](/capturas/CONFIGURACIONREDMYSQL.png)

# Instalación MySQL

Ejecutamos el siguiente comando:

```
apt update && apt install mysql-server -y
```

# Asignación de contraseña para root (MySQL)

Por defecto, root no tiene contraseña para acceder a la base de datos, así que hay que hacer lo siguiente. 
Accedemos a la base de datos con:

```
mysql
```

Ejecutamos la siguiente sentencia:

```
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password by 'antonio';
```

La contraseña para root (MySQL) será antonio.

![](/capturas/CONFIGURACIONROOTMYSQL.png)

# Configuración de la base de datos.

Seguidamente, hay que configurar MySQL ejecutando lo siguiente y contestando a las preguntas:

```
mysql_secure_installation
```

![](/capturas/CONFIGURACIONMYSQLPREGUNTAS.png)
![](/capturas/CONFIGURACIONMYSQLPREGUNTAS2.png)


Después tenemos que añadir una configuración para que se pueda acceder a la base de datos desde otra máquina por lo que modificamos el fichero:

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```

Dentro del fichero buscamos la línea

```
bind-address    = 127.0.0.1
```

y la comentamos


![](/capturas/CONFIGURACIONBDAOTRAMAQUINA.png)


Reiniciamos el servicio:

```
systemctl restart mysql
```

# Creación de base de datos y usuario para Prestashop.

Nos conectamos a la base de datos:

```
mysql -u root -p
```

Después ejecutamos estas sentencias:

```
CREATE DATABASE prestashop character set utf8mb4 collate utf8mb4_unicode_ci;
CREATE USER 'prestashop'@'%' IDENTIFIED BY 'Prestashop123!';
GRANT ALL PRIVILEGES ON prestashop.* TO 'prestashop'@'%';
FLUSH PRIVILEGES;
```

La base de datos se llama prestashop, y solo el usuario prestashop con contraseña Prestashop123! tiene acceso a ella. El % después de la @ significa que el usuario se podrá conectar desde cualquier máquina de la red.

![](/capturas/CONFIGURACIONUSUARIOPRESTASHOPMYSQL.png)

# Configuración de la red interna en ambas máquinas.

Ahora configuramos la red interna para que apache y mysql puedan comunicarse sin acceso a internet.

Para ello, añadimos una nueva tarjeta de red al servidor apache, pero esta vez como red interna.

![](/capturas/CONFIGURACIONREDINTERNAAPACHE.png)

Hacemos lo mismo con mysql, sustituyendo el adaptador puente previo por la red interna.

![](/capturas/CONFIGURACIONREDINTERNAMYSQL.png)

Iniciamos las dos máquinas.

# Configuración de IP para "apache" (red interna).

Lo configuramos como en las anteriores veces:

```
nano /etc/netplan/00-installer-config.yaml
```
Y rellenamos la configuración como en enp0s3:

![](/capturas/CONFIGURACIONREDINTERNAAPACHE1.png)

Por último, aplicamos los cambios:

```
netplan apply
```

# Configuración de IP para "mysql" (red interna)

Abrimos el siguiente fichero como administrador:

```
nano /etc/netplan/00-installer-config.yaml
```

Configuramos:

![](/capturas/CONFIGURACIONREDINTERNAMYSQL1.png)


Por último, aplicamos los cambios:

```
netplan apply
```

# Comprobaciones

Para ver que todo esté funcionando correctamente hasta ahora, hacemos ping entre servidor apache y servidor mysql.

También comprobamos que se puede conectar la base de datos mysql desde la máquina apache, por lo que hacemos lo siguiente desde el servidor apache:

```
apt install mysql-client -y
```

Despues conectamos con la base de datos:

```
mysql -u prestashop -p -h 10.20.0.210
```

> Nota: Conectamos con el usuario prestashop y su contraseña(Prestashop123!).  

![](/capturas/CONEXIONMYSQLDESDEAPACHE.png)


# Instalación de Prestashop.

Prestashop requiere más configuración del servidor apache y también necesita más instalaciones de módulos de PHP, así que procederemos a solucionar todo esto para poder instalar Prestashop (Todas estas instalaciones serán en servidor apache).

# Instalación de otros módulos de PHP.

```
apt install php7.4-curl php7.4-gd php7.4-intl php7.4-mbstring php7.4-xml php7.4-zip php7.4-apcu php7.4-memcached php7.4-memcache -y
```

Reiniciamos apache2:

```
systemctl reload apache2
```

# Configuración adicional de Apache.

Tenemos que activar el módulo mod_rewrite de Apache y reiniciar el servicio:

```
a2enmod rewrite
systemctl restart apache2
```

![](/capturas/CONFIGURACIONADICIONALAPACHE.png)

Modificamos la configuración del siguente fichero:

```
nano /etc/apache2/apache2.conf
```

Dentro del fichero buscamos "AllowOverride" y cambiamos None por All:

![](/capturas/CONFIGURACIONFICHEROAPACHE2.CONF.png)

Recargamos la configuracion de apache2:

```
systemctl reload apache2
```

# Descarga de ficheros de Prestashop (en apache).

Ahora podemos descargar Prestashop:

```
wget https://download.prestashop.com/download/releases/prestashop_1.7.8.7.zip
```

![](/capturas/INSTALACIONPRESTASHOP.png)

Lo descomprimimos en /var/www/html:

> Nota: Si ni tenemos unzip instalados procedemos a intalarlo antes.

```
apt install unzip -y
unzip prestashop_1.7.8.7.zip -d /var/www/html
```

![](/capturas/DESCOMPRIMEPRESTASHOP.png)

Ahora vamos a /var/www/htl y vemos los ficheros:

![](/capturas/VISTAFICHEROSPRESTASHOP.png)

# Ejecución del instalador web.

Desde el navegador ponemos nuestra ip del servidor apache y veremos que cargará una página de Prestashop:

![](/capturas/CAPTURAINSTALADORPRESTASHOP.png)

Para que no salga más el error señalado arriba, tenemos que cambiar los permisos de los ficheros o cambiarlos de propietario.

```
chmod -R 777 /var/www/html
```

Al cambiar los permisos, al acceder otra vez a la web veremos que en la pantalla aparece como se instala Prestashop:

![](/capturas/INSTALADORPRESTASHOP.png)

# Formulario de instalación

La primera pantalla es para seleccionar el idioma.


![](/capturas/IDIOMAPRESTASHOP.png)

Aceptamos los acuerdos de licencia:

![](/capturas/ACUERDOSDELICNECIAPRESTASHOP.png)

Aceptamos la copatibilidad del sistema:

![](/capturas/COMPATIBILIDADELSISTEMA.png)

Información de la tienda:

![](/capturas/ASISTENTEDEINSTALACIONPRESTASHOP.png)

Configuramos la base de datos. Rellenamos con los parámetros antes mencionados. Se puede probar la conexión antes de seguir.

> Nota: Recordad que la ip que ponemos en el asistente de instalación será la ip de nuestro máquina mysql

![](/capturas/ASISTENTEINSTALACIONBDPRESTASHOP.png)

Instalación de la tienda.

![](/capturas/ASISTENTEINSTALACIONFINAL.png)

Finalización

![](/capturas/FINALIZACION.png)

# Configuración para después de la instalación.

Eliminamos la carpeta install.

```
rm -rf /var/www/html/install
```

Cambiamos los permisos por los adecuados, es decir, los archivos solo se pueden leer y escribir unicamente por el usuario "www-data" (es el usuario que usa apache):

```
cd /var/www/html/
find . -type f -exec chmod 644 -- {} +
find . -type d -exec chmod 755 -- {} +
chown -R www-data:www-data .
```

# Acceso a la tienda

Para acceder a la tienda, escribimos la ip de apache seguido de /admin y nos redirigirá a una pantalla de login:

![](/capturas/CAPTURAADMIN.png)

Accedemos con los datos qe hemos rellenado e el formulario de instalación.

![](/capturas/CAPTURABIENVENIDA.png)


# Configuraciones de nuestra tienda.

## Zona Horaria

Vamos a Personalizar > Internacional > Localización > Configuración > Zona Horaria

![](/capturas/ZONAHORARIA.png)

## Cambiar logo

Ir a Personalizar > Diseño > Tema y Logotipo

![](/capturas/TEMAYLOGOTIPO.png)

Si acudimos a nuestra web, vemos que hemos añadido una foto de cabecera.

![](/capturas/LOGOTIENDA.png)

## Cambiar parámetros de seguridad

Ir a Configurar > Parámetros de la tienda > Configuración

Aquí podemos configurar varios parametros, entre ellos de seguridad.

![](/capturas/PARAMETROSDECONFIGURACION.png)


## Respaldar base de datos

Se puede realizar una copia de la base de datos desde la opción Configurar > Parámetros Avanzados > Base de datos > Respaldar BD

![](/capturas/RESPALDARBD.png)

Aceptamos la advertencia, se hace la copia y la descargamos.

![](/capturas/RESPALDOBD.png)


# Referencias

- [Linuxize - Configurar IP estática en Ubuntu](https://linuxize.com/post/how-to-configure-static-ip-address-on-ubuntu-20-04/)
- [DigitalOcean - Instalar LAMP](https://www.digitalocean.com/community/tutorials/how-to-install-linux-apache-mysql-php-lamp-stack-on-ubuntu-20-04#step-6-%E2%80%94-testing-database-connection-from-php-(optional))
- [Pleets Blog - Establecer contraseña para root (MySQL)](https://blog.pleets.org/article/soluci%C3%B3n-al-error-set-password-has-no-significance-for-user)
- [StackOverflow - Conexión MySQL Error 111](https://stackoverflow.com/questions/1420839/cant-connect-to-mysql-server-error-111)
- [DigitalOcean - Crear BD y usuario MySQL](https://www.digitalocean.com/community/tutorials/crear-un-nuevo-usuario-y-otorgarle-permisos-en-mysql-es)
- [Cómo instalar - Prestashop](https://comoinstalar.me/como-instalar-prestashop-en-ubuntu-20-04-lts/)
- [DigitalOcean - Instalar PHP 7.4](https://www.digitalocean.com/community/tutorials/how-to-install-php-7-4-and-set-up-a-local-development-environment-on-ubuntu-20-04)




















