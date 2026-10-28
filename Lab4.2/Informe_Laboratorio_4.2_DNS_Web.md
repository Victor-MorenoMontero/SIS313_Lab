# Informe de Laboratorio 4.2: Servidor DNS Primario e Integración con Servicio Web

> **Nota para completar el informe:** sustituye los campos entre corchetes por tus datos reales e inserta las capturas indicadas donde aparecen los marcadores. Las capturas deben mostrar resultados obtenidos durante tu práctica; no incluyas resultados que no hayas verificado.

## Datos generales

- **Universidad:** Universidad San Francisco Xavier de Chuquisaca
- **Asignatura:** Infraestructura, Plataformas Tecnológicas y Redes (SIS313)
- **Docente:** Ing. Marcelo Quispe Ortega
- **Semestre:** 2/2026
- **Estudiante(s):** [Nombre completo]
- **Fecha de realización:** [Fecha]

## 1. Introducción

En esta práctica se trabaja con una infraestructura virtual compuesta por máquinas Ubuntu Server 24.04 LTS para implementar un servidor DNS primario mediante BIND9 y un servidor web mediante Nginx. El propósito es configurar una zona DNS directa, registrar nombres y alias, comprobar la resolución de nombres y acceder a una página web usando un dominio en lugar de depender únicamente de una dirección IP.

La guía contempla una práctica individual con tres máquinas virtuales y una práctica grupal en la que los integrantes despliegan servicios DNS y web accesibles desde la red del laboratorio.

## 2. Objetivos

### 2.1. Objetivo general

Configurar y comprobar un servidor DNS primario con BIND9 e integrarlo con un servicio web Nginx dentro de una red de laboratorio.

### 2.2. Objetivos específicos

- Asignar direcciones IP estáticas a las máquinas virtuales.
- Configurar la comunicación de la red interna y, en el servidor DNS, el reenvío de paquetes cuando corresponda.
- Instalar BIND9 y definir una zona DNS directa.
- Crear registros SOA, NS, A y CNAME.
- Validar la configuración DNS con `named-checkconf`, `named-checkzone`, `dig` y `nslookup`.
- Instalar Nginx y configurar un sitio web estático asociado al dominio.
- Verificar el acceso al sitio desde el cliente y, cuando se configure el reenvío de puertos, desde la PC anfitriona.
- En la práctica grupal, configurar además la zona inversa y comprobar el acceso desde las PCs anfitrionas.

## 3. Materiales y entorno de trabajo

- PC anfitriona con Oracle VirtualBox.
- Tres máquinas virtuales con Ubuntu Server 24.04 LTS para la práctica individual.
- BIND9, `bind9utils` y herramientas de consulta DNS.
- Nginx para publicar una página HTML.
- Terminal de Ubuntu Server y navegador web en el equipo anfitrión.
- Red interna de VirtualBox denominada, por ejemplo, `Red_Lab4_2`.

### 3.1. Topología de la práctica individual

La guía propone la siguiente distribución:

| Máquina virtual | Hostname | Función | Dirección IP interna |
|---|---|---|---|
| `Lab4.2-DNS` | `dns` | Servidor DNS BIND9 | `192.168.10.2/29` |
| `Lab4.2-Web` | `web` | Servidor web Nginx | `192.168.10.3/29` |
| `Lab4.2-Client` | `client` | Cliente de pruebas | `192.168.10.4/29` |

La máquina DNS utiliza dos adaptadores: NAT para acceso a Internet y una interfaz conectada a la red interna. Las máquinas Web y Client se conectan a esa red interna.

**[INSERTAR IMAGEN 1 AQUÍ: Diagrama de la topología o captura de VirtualBox donde se aprecien las tres VMs y sus adaptadores de red.]**

*Figura 1. Topología virtual y conexión de las máquinas del laboratorio.*

## 4. Desarrollo de la práctica individual

### 4.1. Ejercicio 1: Configuración de red estática

Se configuraron las direcciones IP de la red interna mediante Netplan. En el servidor DNS, la interfaz NAT mantiene la configuración DHCP y la interfaz interna utiliza `192.168.10.2/29`.

Archivo de referencia: `/etc/netplan/50-cloud-init.yaml`.

Ejemplo para la VM DNS, ajustando los nombres de las interfaces a los que existan realmente en la máquina:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.10.2/29
      nameservers:
        addresses:
          - 8.8.8.8
```

En la VM Web se asignó `192.168.10.3/29` y en la VM Client `192.168.10.4/29`. Ambas utilizaron `192.168.10.2` como servidor DNS y como puerta de enlace, según la configuración indicada en la guía.

Después de editar cada archivo, se aplicaron los cambios:

```bash
sudo netplan apply
ip address
ip route
```

![Captura de `ip address` en la VM DNS donde se vea la IP interna `192.168.10.2`.](images/DNS_IP.png)

*Figura 2. Verificación de la dirección IP del servidor DNS.*

![Captura de `ip address` en la VM Web y/o Client mostrando las IP asignadas.](images/WEB_IP.png)

*Figura 3. Direccionamiento de las máquinas Web y Client.*

#### Reenvío de paquetes en el servidor DNS

La guía indica habilitar el reenvío IPv4 en `/etc/sysctl.conf`:

```bash
sudo nano /etc/sysctl.conf
```

Se verificó que la siguiente línea estuviera activa:

```text
net.ipv4.ip_forward=1
```

Luego se aplicó la configuración y se preparó la regla NAT:

```bash
sudo sysctl -p
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

**[INSERTAR IMAGEN 4 AQUÍ: Captura de `sysctl -p` mostrando el reenvío IPv4 activo y/o de `iptables -t nat -L -n -v` mostrando la regla NAT.]**

*Figura 4. Habilitación del reenvío de paquetes y configuración NAT.*

### 4.2. Ejercicio 2: Instalación y configuración de BIND9

En la VM `Lab4.2-DNS` se instalaron los paquetes necesarios:

```bash
sudo apt update
sudo apt install bind9 bind9utils bind9-doc -y
```

Se declaró la zona maestra `lab42.local` en `/etc/bind/named.conf.local`:

```bind
zone "lab42.local" {
    type master;
    file "/etc/bind/db.lab42.local";
};
```

A continuación, se creó el archivo de zona a partir de la plantilla y se editó:

```bash
sudo cp /etc/bind/db.local /etc/bind/db.lab42.local
sudo nano /etc/bind/db.lab42.local
```

Contenido de referencia de la zona:

```bind
$TTL    604800
@       IN      SOA     dns.lab42.local. admin.lab42.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

@       IN      NS      dns.lab42.local.
dns     IN      A       192.168.10.2
www     IN      A       192.168.10.3
web     IN      CNAME   www.lab42.local.
```

Los registros cumplen las siguientes funciones:

- **SOA:** identifica la autoridad principal de la zona y sus parámetros.
- **NS:** especifica el servidor de nombres autoritativo.
- **A:** relaciona `dns.lab42.local` y `www.lab42.local` con sus direcciones IPv4.
- **CNAME:** crea el alias `web.lab42.local` que apunta a `www.lab42.local`.

Se verificó la sintaxis de la configuración y de la zona:

```bash
sudo named-checkconf
sudo named-checkzone lab42.local /etc/bind/db.lab42.local
```

Después se reinició y consultó el estado del servicio:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

La guía también propone comprobar el puerto DNS con:

```bash
sudo netstat -tulnp | grep named
```

Si `netstat` no está instalado, se puede usar `sudo ss -tulnp | grep ':53'` para revisar los sockets que escuchan en el puerto 53.

**[INSERTAR IMAGEN 5 AQUÍ: Captura de `named-checkzone lab42.local /etc/bind/db.lab42.local` con el resultado `OK`.]**

*Figura 5. Validación de la zona directa de BIND9.*

**[INSERTAR IMAGEN 6 AQUÍ: Captura de `systemctl status bind9` mostrando el estado real del servicio.]**

*Figura 6. Estado del servicio BIND9.*

### 4.3. Ejercicio 3: Configuración del servidor web Nginx

En la VM `Lab4.2-Web` se instaló y habilitó Nginx:

```bash
sudo apt update
sudo apt install nginx -y
sudo systemctl enable --now nginx
```

Se creó el archivo de configuración del sitio:

```bash
sudo nano /etc/nginx/sites-available/lab42.local
```

Configuración de referencia:

```nginx
server {
    listen 80;
    server_name www.lab42.local lab42.local;

    root /var/www/lab42.local;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Se creó el directorio del sitio, se habilitó el bloque y se retiró el sitio predeterminado, tal como indica la guía:

```bash
sudo mkdir -p /var/www/lab42.local
sudo ln -s /etc/nginx/sites-available/lab42.local /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
```

Se añadió una página HTML de prueba:

```bash
sudo bash -c 'echo "<h1>Bienvenido a www.lab42.local</h1>" > /var/www/lab42.local/index.html'
sudo bash -c 'echo "<p>Servido desde la VM Web (192.168.10.3)</p>" >> /var/www/lab42.local/index.html'
```

Por último, se comprobó la sintaxis y se reinició el servicio:

```bash
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl status nginx
```

**[INSERTAR IMAGEN 7 AQUÍ: Captura de `nginx -t` mostrando que la sintaxis es correcta.]**

*Figura 7. Validación de la configuración de Nginx.*

**[INSERTAR IMAGEN 8 AQUÍ: Captura del navegador o de `curl` mostrando la página de bienvenida de `www.lab42.local`.]**

*Figura 8. Página web publicada desde la VM Web.*

### 4.4. Ejercicio 4: Pruebas de resolución DNS y acceso web

Desde la VM Client se realizaron las siguientes pruebas, usando el servidor DNS interno:

```bash
dig @192.168.10.2 www.lab42.local
nslookup www.lab42.local 192.168.10.2
dig @192.168.10.2 web.lab42.local
curl http://www.lab42.local
```

La comprobación debía confirmar que `www.lab42.local` resolviera a `192.168.10.3`, que el alias `web.lab42.local` apuntara al nombre web y que `curl` devolviera el HTML de la página.

**[INSERTAR IMAGEN 9 AQUÍ: Captura de `dig @192.168.10.2 www.lab42.local` donde se vea la respuesta y la dirección IP.]**

*Figura 9. Resolución directa del nombre del servidor web.*

**[INSERTAR IMAGEN 10 AQUÍ: Captura de `dig @192.168.10.2 web.lab42.local` o de `nslookup` donde se aprecie el alias CNAME.]**

*Figura 10. Comprobación del alias DNS.*

**[INSERTAR IMAGEN 11 AQUÍ: Captura de `curl http://www.lab42.local` mostrando el HTML de bienvenida.]**

*Figura 11. Acceso al contenido web desde el cliente.*

#### Acceso desde la PC anfitriona (opcional)

La guía propone editar el archivo `hosts` del equipo anfitrión si se desea usar el dominio desde el navegador:

- Windows: `C:\Windows\System32\drivers\etc\hosts`
- Linux/macOS: `/etc/hosts`

Ejemplo de entradas:

```text
127.0.0.1   www.lab42.local
127.0.0.1   lab42.local
```

Si se configuró el reenvío del puerto 8080 del anfitrión al puerto 80 de la VM Web, se puede probar `http://127.0.0.1:8080`. Esta prueba comprueba el acceso web a través del reenvío; por sí sola no demuestra que el anfitrión esté consultando el servidor BIND9.

**[INSERTAR IMAGEN 12 AQUÍ: Captura del navegador del anfitrión mostrando el sitio por el puerto 8080, si se configuró el reenvío.]**

*Figura 12. Acceso al servidor web desde el equipo anfitrión.*

## 5. Desarrollo de la práctica grupal

> Completa esta sección solo con las tareas realizadas por tu grupo. Cambia el dominio y las IP de ejemplo por los valores realmente asignados en el aula.

### 5.1. Topología y asignación de roles

La práctica grupal contempla dos integrantes: uno se encarga del servidor DNS con BIND9 y otro del servidor web con Nginx. Las máquinas virtuales se conectan mediante el modo de red **Puente (Bridge)**, con direcciones IP disponibles en la red del laboratorio.

| Rol | Responsabilidad |
|---|---|
| Integrante 1: DNS | Configurar BIND9, zona directa, zona inversa y pruebas de resolución. |
| Integrante 2: Web | Configurar Nginx, crear el sitio HTML y asociarlo al dominio del grupo. |

- **Dominio elegido:** [Dominio único del grupo]
- **IP del servidor DNS:** [IP real]
- **IP del servidor web:** [IP real]
- **Integrantes:** [Nombres completos]

**[INSERTAR IMAGEN 13 AQUÍ: Captura de la configuración de red en modo Puente o diagrama de las VMs del grupo con sus IP reales.]**

*Figura 13. Topología de la práctica grupal.*

### 5.2. Configuración de la zona directa

Se configuró una zona directa para el dominio del grupo, con registros SOA, NS, A y CNAME. La guía presenta como ejemplo `los-pepes.red`; se debe usar el dominio propio del grupo y sustituir las direcciones de ejemplo por las direcciones reales.

Comandos de validación:

```bash
sudo named-checkconf
sudo named-checkzone DOMINIO_DEL_GRUPO /etc/bind/db.DOMINIO_DEL_GRUPO
sudo systemctl restart bind9
```

**[INSERTAR IMAGEN 14 AQUÍ: Captura del archivo de zona directa y/o del resultado satisfactorio de `named-checkzone`, con el dominio real del grupo.]**

*Figura 14. Configuración y validación de la zona directa grupal.*

### 5.3. Configuración de la zona inversa

Para la práctica grupal, la guía solicita crear una zona inversa correspondiente al segmento asignado. Como ejemplo para `192.168.5.0/24`, se utiliza la zona `5.168.192.in-addr.arpa` y registros PTR que relacionan los últimos octetos con los nombres DNS.

Ejemplo de validación para el rango de muestra:

```bash
sudo named-checkzone 5.168.192.in-addr.arpa /etc/bind/db.192.168.5
```

El nombre de la zona y la ruta del archivo deben adaptarse a la red realmente asignada.

**[INSERTAR IMAGEN 15 AQUÍ: Captura del archivo de zona inversa y del resultado `OK` de `named-checkzone`.]**

*Figura 15. Configuración y validación de la zona inversa.*

### 5.4. Configuración de Nginx y página del grupo

El servidor web debe publicar una página HTML estática que incluya, como mínimo, el nombre del grupo, los integrantes y un mensaje o imagen de bienvenida. El bloque `server` de Nginx debe tener el dominio del grupo en `server_name` y la ruta correcta del sitio en `root`.

**[INSERTAR IMAGEN 16 AQUÍ: Captura del archivo HTML o del Virtual Host de Nginx del grupo.]**

*Figura 16. Página HTML y configuración del servidor web grupal.*

### 5.5. Pruebas desde las PCs anfitrionas

Con las PCs configuradas para consultar la IP del servidor DNS del grupo, se realizaron las pruebas indicadas en la guía:

```bash
nslookup www.DOMINIO_DEL_GRUPO
dig www.DOMINIO_DEL_GRUPO
nslookup IP_DEL_SERVIDOR_WEB
dig -x IP_DEL_SERVIDOR_WEB
```

Finalmente, se abrió en el navegador:

```text
http://www.DOMINIO_DEL_GRUPO
```

Las pruebas deben mostrar la resolución directa del nombre web, la resolución inversa a través de PTR y la página HTML servida por Nginx.

**[INSERTAR IMAGEN 17 AQUÍ: Captura de `nslookup` o `dig` para la resolución directa desde una PC anfitriona.]**

*Figura 17. Resolución directa desde la red del laboratorio.*

**[INSERTAR IMAGEN 18 AQUÍ: Captura de `dig -x IP_DEL_SERVIDOR_WEB` o `nslookup IP_DEL_SERVIDOR_WEB` mostrando el nombre devuelto por la zona inversa.]**

*Figura 18. Resolución inversa del servidor web.*

**[INSERTAR IMAGEN 19 AQUÍ: Captura del navegador de una PC anfitriona mostrando el sitio web del grupo por su dominio.]**

*Figura 19. Acceso al sitio web grupal mediante el dominio DNS.*

## 6. Resultados

Completa esta tabla a partir de las salidas reales obtenidas. No marques una prueba como satisfactoria si no se ejecutó o si el resultado no coincide.

| Prueba | Resultado observado | Estado |
|---|---|---|
| IP estáticas en las VMs individuales | [Anotar IP comprobadas] | [Correcto/Pendiente] |
| `named-checkconf` | [Salida real] | [Correcto/Pendiente] |
| `named-checkzone` de la zona directa individual | [Salida real] | [Correcto/Pendiente] |
| Estado de BIND9 | [Salida real] | [Correcto/Pendiente] |
| `nginx -t` | [Salida real] | [Correcto/Pendiente] |
| Resolución de `www.lab42.local` | [IP devuelta] | [Correcto/Pendiente] |
| Resolución del alias `web.lab42.local` | [Salida real] | [Correcto/Pendiente] |
| Acceso web desde la VM Client | [Respuesta real] | [Correcto/Pendiente] |
| Zona directa grupal | [Salida real] | [Correcto/Pendiente] |
| Zona inversa grupal | [Nombre devuelto] | [Correcto/Pendiente] |
| Acceso desde PC anfitriona | [URL y resultado] | [Correcto/Pendiente] |

## 7. Problemas encontrados y soluciones

Describe aquí únicamente los inconvenientes que realmente ocurrieron durante la práctica. Para cada uno, registra el síntoma, la causa identificada y la solución aplicada.

| Problema observado | Causa identificada | Solución aplicada |
|---|---|---|
| [Ej. el nombre DNS no resolvía] | [Causa comprobada] | [Acción aplicada] |
| [Ej. Nginx no mostraba la página] | [Causa comprobada] | [Acción aplicada] |

Comandos útiles de diagnóstico que pueden emplearse según el problema:

```bash
ip address
ip route
sudo named-checkconf
sudo named-checkzone lab42.local /etc/bind/db.lab42.local
sudo systemctl status bind9
sudo nginx -t
sudo systemctl status nginx
```

## 8. Conclusiones

[Redacta las conclusiones a partir de los resultados comprobados. Indica si se logró configurar BIND9, si la zona DNS resolvió los nombres y alias, si Nginx publicó el sitio y si las pruebas grupales de resolución inversa y acceso desde las PCs anfitrionas funcionaron.]

## 9. Recomendaciones

- Verificar los nombres reales de las interfaces de red antes de editar Netplan.
- Revisar la sintaxis de BIND9 y Nginx antes de reiniciar los servicios.
- Confirmar que las IP configuradas coincidan con los registros DNS.
- Comprobar que los clientes consulten el servidor DNS correcto.
- En la práctica grupal, coordinar el dominio y las IP para evitar conflictos dentro del aula.
- Capturar las pruebas con el comando y su salida legibles, evitando recortar el dato que demuestra el resultado.

## 10. Anexo de evidencias

Antes de entregar el informe, comprueba que las capturas estén insertadas y numeradas en el lugar indicado. Como mínimo, incluye evidencias de la configuración de red, validación de BIND9, validación de Nginx, resolución DNS, acceso web y, en la práctica grupal, resolución inversa y acceso desde una PC anfitriona.

---

**Fin del informe.**
