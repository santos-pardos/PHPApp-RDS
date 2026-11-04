# Instalar PHP_AWS_App en una instancia EC2

Esta guía instala la aplicación PHP del laboratorio de AWS Academy Cloud Foundations en una instancia EC2 con Amazon Linux 2. La página principal muestra el ID de instancia y la zona de disponibilidad; el menú también incluye una prueba de carga y una página de conexión a RDS.

> La aplicación es antigua. La página de RDS usa las funciones PHP `mysql_*`, eliminadas en PHP 7. En versiones actuales de PHP, la web puede arrancar pero la parte de RDS necesitará adaptar el código a `mysqli` o PDO. No se recomienda instalar PHP antiguo en una instancia expuesta a Internet.

## 1. Preparar la instancia

En EC2, lanza una instancia **Amazon Linux 2** en una subred pública y asígnale una dirección IPv4 pública. En su grupo de seguridad, permite:

- **HTTP, TCP 80**, desde tu IP pública para probar el sitio. Para un laboratorio compartido puedes abrirlo a `0.0.0.0/0` temporalmente.
- **SSH, TCP 22**, solo desde tu IP, si vas a conectarte por SSH. También puedes usar EC2 Instance Connect si está habilitado.

Comprueba que la subred tenga una ruta a un Internet Gateway y que la instancia tenga salida a Internet para descargar el ZIP.

## 2. Instalar Apache, PHP y herramientas

Conéctate a la instancia por SSH o EC2 Instance Connect y ejecuta:

```bash
sudo yum update -y
sudo yum install -y httpd php php-cli php-curl unzip wget procps-ng
```

`php-curl` permite consultar los metadatos de EC2 mediante IMDSv2; `procps-ng` proporciona `vmstat`, que usa la página de carga de CPU.

## 3. Descargar y desplegar la aplicación

Descarga el ZIP publicado para el laboratorio y extráelo en el directorio web de Apache:

```bash
cd /tmp
wget -O lab-app.zip 'https://aws-tc-largeobjects.s3.amazonaws.com/AWS-TC-AcademyACF/acf-lab3-vpc/lab-app.zip'
sudo unzip -o lab-app.zip -d /var/www/html/
sudo chown -R apache:apache /var/www/html
```

El archivo `index.php` debe quedar directamente en `/var/www/html/index.php`. No debe quedar dentro de una carpeta adicional llamada `lab-app`.

## 4. Iniciar Apache y comprobar la web

```bash
sudo systemctl enable --now httpd
sudo systemctl status httpd --no-pager
curl -I http://localhost/
```

Abre en el navegador:

```text
http://IP-PUBLICA-DE-LA-EC2/
```

Deberías ver el logo de AWS, el ID de la instancia y la zona de disponibilidad, además de los enlaces **Load Test** y **RDS**.

Si Apache no arranca o la página no carga, revisa los registros:

```bash
sudo tail -n 60 /var/log/httpd/error_log
```

Comprueba también que el grupo de seguridad permite TCP 80 y que estás usando la IP pública de la instancia.

## 5. Probar la carga de CPU

Abre el enlace **Load Test**. La página muestra la carga actual y ejecuta la rutina de prueba del laboratorio. Al terminar, cierra esa pestaña para detener las peticiones repetidas.

## 6. Configurar la parte de RDS (opcional)

La aplicación espera una base de datos MySQL con una tabla `address`. Para probarla:

1. Crea una base de datos MySQL en RDS, por ejemplo con nombre `lab`.
2. En el grupo de seguridad de RDS, permite entrada **TCP 3306** desde el grupo de seguridad de la instancia EC2. No abras el puerto a todas las IP.
3. En el navegador, selecciona **RDS** e introduce el endpoint, el nombre de la base de datos, el usuario y la contraseña.
4. El instalador intenta cargar `sql/addressbook.sql` y guardar los datos de conexión en `rds.conf.php`.

**Compatibilidad:** en el ZIP, `rds-read-data.php` y `rds-write-config.php` llaman a `mysql_connect`, `mysql_query` y otras funciones `mysql_*`. PHP 7 y versiones posteriores ya no las incluyen. Si aparece un error como `Call to undefined function mysql_connect()`, la descarga y despliegue sí se han realizado, pero la aplicación de RDS debe actualizarse a `mysqli` o PDO antes de poder usarla con ese PHP.

## Instalación automática con User Data

También puedes pegar este script en **Advanced details → User data** al lanzar la instancia. Está pensado para Amazon Linux 2:

```bash
#!/bin/bash
yum update -y
yum install -y httpd php php-cli php-curl unzip wget procps-ng
cd /tmp
wget -O lab-app.zip 'https://aws-tc-largeobjects.s3.amazonaws.com/AWS-TC-AcademyACF/acf-lab3-vpc/lab-app.zip'
unzip -o lab-app.zip -d /var/www/html/
chown -R apache:apache /var/www/html
systemctl enable --now httpd
```

Tras el primer arranque, espera a que EC2 indique que las comprobaciones de estado han pasado y visita la IP pública.

## Limpieza

Cuando termines, elimina la instancia EC2 y cualquier recurso RDS que hayas creado para la práctica. Así evitarás cargos posteriores.
