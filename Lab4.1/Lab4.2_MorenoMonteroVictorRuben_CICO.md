# Informe de Laboratorio 4.1: Plataforma HA, Balanceo de Carga y Monitoreo

**Universidad San Francisco Xavier de Chuquisaca**

| | |
| - | - |
| **Asignatura** | Infraestructura, Plataformas Tecnológicas y Redes (SIS313) |
| **Docente** | Ing. Marcelo Quispe Ortega |
| **Semestre** | 2/2026 |
| **Estudiante(s)** | _(Nombre completo)_ |
| **Grupo / N°** | _(N)_ |
| **Fecha** | _(DD/MM/2026)_ |

> **Cómo usar este informe:** cada recuadro `📸 CAPTURA` indica qué imagen debe insertarse. Guarda tus capturas en una carpeta `img/` junto a este archivo y reemplaza la ruta de ejemplo (`img/xx.png`) por el nombre real de tu archivo.

---

## Índice

1. [Introducción](#1-introducción)
2. [Objetivos](#2-objetivos)
3. [Arquitectura de la solución](#3-arquitectura-de-la-solución)
4. [Preparación del entorno virtual](#4-preparación-del-entorno-virtual)
5. [Ejercicio 1: Proxy inverso y balanceo de carga](#5-ejercicio-1-proxy-inverso-y-balanceo-de-carga-vm-lab41-proxy)
6. [Ejercicio 2: Servidores de aplicaciones](#6-ejercicio-2-servidores-de-aplicaciones-vm-lab41-apps)
7. [Ejercicio 3: Servidor de base de datos](#7-ejercicio-3-servidor-de-base-de-datos-vm-lab41-db)
8. [Ejercicio 4: Lanzamiento de instancias con PM2](#8-ejercicio-4-lanzamiento-de-instancias-con-pm2-vm-lab41-apps)
9. [Ejercicio 5: Servicios de monitoreo](#9-ejercicio-5-servicios-de-monitoreo-vm-lab41-proxy)
10. [Pruebas de alta disponibilidad (failover)](#10-pruebas-de-alta-disponibilidad-failover)
11. [Problemas encontrados y soluciones](#11-problemas-encontrados-y-soluciones)
12. [Conclusiones y reflexiones](#12-conclusiones-y-reflexiones)
13. [Anexos](#13-anexos)

---

## 1. Introducción

_(Redacta en 1 o 2 párrafos el propósito de la práctica. Sugerencia: explica qué es la alta disponibilidad, por qué se separan los servicios (proxy, aplicaciones y base de datos) y qué aporta el monitoreo en una plataforma real.)_

Este informe documenta la implementación de una arquitectura web de **Alta Disponibilidad (HA)** compuesta por:

- Un **proxy inverso y balanceador de carga** (Nginx).
- Dos **instancias de una API RESTful** en Node.js gestionadas con **PM2**.
- Una **base de datos centralizada** (MariaDB).
- Un sistema de **monitoreo** (Prometheus + Node Exporter + Grafana).

## 2. Objetivos

- Implementar una arquitectura web de Alta Disponibilidad con balanceo de carga, segregación de servicios y monitoreo de rendimiento.
- Configurar Nginx como proxy inverso para balancear el tráfico entre dos servidores de aplicaciones.
- Desplegar dos instancias de aplicación Node.js gestionadas por PM2 en modo producción.
- Instalar y configurar una base de datos centralizada MariaDB y asegurar su acceso.
- Integrar Prometheus y Grafana para medir la disponibilidad y el rendimiento de la plataforma.

---

## 3. Arquitectura de la solución

### 3.1 Diagrama de arquitectura

> 📸 **CAPTURA / DIAGRAMA 01:** Diagrama de la topología (puede hacerse en draw.io, Excalidraw, Mermaid, etc.). Debe mostrar: PC anfitriona → NAT/Port Forwarding → VM Proxy → Red interna → VM Apps (App 1 y App 2) y VM DB, con sus IPs.
>
> ![Diagrama de arquitectura](img/01-diagrama-arquitectura.png)
> _Figura 1. Arquitectura de red y roles de cada VM._

### 3.2 Asignación de IPs y roles

La red interna utiliza el segmento `192.168.10.0/29`.

| VM | Hostname | Rol | Interfaces y conexión | IP interna |
| - | - | - | - | - |
| `Lab4.1-Proxy` | `proxy` | Proxy + Monitoreo | NAT (Internet) + Red interna | `192.168.10.2` |
| `Lab4.1-Apps` | `apps` | Aplicaciones (App 1 y App 2) | Red interna | `192.168.10.3` |
| `Lab4.1-DB` | `db` | Base de datos | Red interna | `192.168.10.4` |

La interfaz interna de `Lab4.1-Proxy` actúa como **gateway** de las VMs `Apps` y `DB` para su salida a Internet.

### 3.3 Flujo de una petición

```
Cliente → Nginx (proxy:80) ─┬→ App 1 (apps:3001) ─┐
                            └→ App 2 (apps:3002) ─┴→ MariaDB (db:3306)
```

---

## 4. Preparación del entorno virtual

### 4.1 Creación de la red interna en VirtualBox

Se creó la red `Red_Lab4_1` desde **Herramientas → Redes → Crear**.

> 📸 **CAPTURA 02:** Ventana de VirtualBox mostrando la red `Red_Lab4_1` creada.
>
> ![Red interna creada](img/02-virtualbox-red-interna.png)
> _Figura 2. Red interna `Red_Lab4_1` en VirtualBox._

### 4.2 Configuración de adaptadores de red por VM

| VM | Adaptador 1 | Adaptador 2 |
| - | - | - |
| `Lab4.1-Proxy` | NAT | Red interna (`Red_Lab4_1`) |
| `Lab4.1-Apps` | Red interna (`Red_Lab4_1`) | — |
| `Lab4.1-DB` | Red interna (`Red_Lab4_1`) | — |

> 📸 **CAPTURA 03:** Configuración de red (Adaptadores 1 y 2) de la VM `Lab4.1-Proxy`.
>
> ![Adaptadores VM Proxy](img/03-adaptadores-proxy.png)
> _Figura 3. Adaptadores de red de `Lab4.1-Proxy`._

> 📸 **CAPTURA 04:** Configuración de red (Adaptador 1) de las VMs `Lab4.1-Apps` y `Lab4.1-DB`.
>
> ![Adaptadores VM Apps y DB](img/04-adaptadores-apps-db.png)
> _Figura 4. Adaptadores de red de `Lab4.1-Apps` y `Lab4.1-DB`._

### 4.3 Reenvío de puertos (Port Forwarding) en `Lab4.1-Proxy`

| Nombre | Protocolo | IP Host | Puerto Host | IP Invitado | Puerto Invitado | Propósito |
| - | - | - | - | - | - | - |
| SSH | TCP | 127.0.0.1 | 2222 | 10.0.2.15 | 22 | Acceso remoto |
| PROXY | TCP | 127.0.0.1 | 80 | 10.0.2.15 | 80 | Acceso web (Nginx) |
| GRAFANA | TCP | 127.0.0.1 | 8080 | 10.0.2.15 | 3000 | Monitoreo web |

> 📸 **CAPTURA 05:** Tabla de reglas de reenvío de puertos en VirtualBox (Adaptador 1 → Avanzado → Reenvío de puertos).
>
> ![Port forwarding](img/05-port-forwarding.png)
> _Figura 5. Reglas de reenvío de puertos de la VM Proxy._

---

## 5. Ejercicio 1: Proxy inverso y balanceo de carga (VM `Lab4.1-Proxy`)

### 5.1 Configuración de IP estática

Se editó el archivo `/etc/netplan/50-cloud-init.yaml`:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

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

```bash
sudo netplan apply
```

> 📸 **CAPTURA 06:** Archivo Netplan de la VM Proxy (`cat /etc/netplan/50-cloud-init.yaml`).
>
> ![Netplan Proxy](img/06-netplan-proxy.png)
> _Figura 6. Configuración Netplan de `Lab4.1-Proxy`._

> 📸 **CAPTURA 07:** Salida del comando `ip addr` en la VM Proxy mostrando `enp0s3` (NAT) y `enp0s8` con `192.168.10.2/29`.
>
> ![ip addr Proxy](img/07-ip-addr-proxy.png)
> _Figura 7. Interfaces de red de `Lab4.1-Proxy`._

### 5.2 Instalación de Nginx

```bash
sudo apt update && sudo apt install nginx -y
```

> 📸 **CAPTURA 08:** Estado de Nginx (`systemctl status nginx`) mostrando el servicio `active (running)`.
>
> ![Estado de Nginx](img/08-nginx-status.png)
> _Figura 8. Nginx instalado y en ejecución._

### 5.3 Configuración del balanceador

Se configuró el bloque `upstream` en `/etc/nginx/sites-available/default`:

```nginx
upstream loadbalancer {
    server 192.168.10.3:3001; # App 1
    server 192.168.10.3:3002; # App 2
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://loadbalancer;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

> 📸 **CAPTURA 09:** Contenido del archivo `/etc/nginx/sites-available/default` con el `upstream` y el `server`.
>
> ![Configuración Nginx](img/09-nginx-config.png)
> _Figura 9. Configuración del balanceador de carga en Nginx._

Se validó la sintaxis y se reinició el servicio:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

> 📸 **CAPTURA 10:** Resultado de `sudo nginx -t` (syntax is ok / test is successful).
>
> ![nginx -t](img/10-nginx-test.png)
> _Figura 10. Validación de la configuración de Nginx._

### 5.4 Enrutamiento y NAT

**Reenvío de paquetes en el kernel** (`/etc/sysctl.conf`):

```bash
sudo nano /etc/sysctl.conf
```

```bash
net.ipv4.ip_forward=1
```

```bash
sudo sysctl -p
```

> 📸 **CAPTURA 11:** Línea `net.ipv4.ip_forward=1` en `/etc/sysctl.conf` y salida de `sudo sysctl -p`.
>
> ![ip_forward](img/11-ip-forward.png)
> _Figura 11. Habilitación del reenvío de paquetes IPv4._

**Regla de enmascaramiento (NAT)** y persistencia:

```bash
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

> 📸 **CAPTURA 12:** Reglas NAT activas (`sudo iptables -t nat -L -n -v`) y confirmación de `netfilter-persistent save`.
>
> ![iptables NAT](img/12-iptables-nat.png)
> _Figura 12. Regla MASQUERADE configurada y guardada._

---

## 6. Ejercicio 2: Servidores de aplicaciones (VM `Lab4.1-Apps`)

### 6.1 Hostname (si la VM fue clonada)

```bash
sudo hostnamectl set-hostname apps
```

### 6.2 Configuración de IP estática

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.10.3/29
      routes:
        - to: default
          via: 192.168.10.2
      nameservers:
        addresses:
          - 8.8.8.8
```

```bash
sudo netplan apply
```

> 📸 **CAPTURA 13:** Salida de `ip addr` en la VM Apps con la IP `192.168.10.3/29`.
>
> ![ip addr Apps](img/13-ip-addr-apps.png)
> _Figura 13. Configuración de red de `Lab4.1-Apps`._

> 📸 **CAPTURA 14:** Pruebas de conectividad desde Apps: `ping 192.168.10.2` (proxy) y `ping 8.8.8.8` (salida a Internet a través del gateway).
>
> ![Ping desde Apps](img/14-ping-apps.png)
> _Figura 14. Conectividad de Apps con el proxy y con Internet._

### 6.3 Instalación de Node.js (vía nvm)

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
npm -v
```

> 📸 **CAPTURA 15:** Salida de `node -v` y `npm -v`.
>
> ![Versiones Node y npm](img/15-node-npm-version.png)
> _Figura 15. Versiones instaladas de Node.js y npm._

### 6.4 Instalación de PM2

```bash
npm install pm2@latest -g
pm2 --version
```

> 📸 **CAPTURA 16:** Salida de `pm2 --version`.
>
> ![Versión PM2](img/16-pm2-version.png)
> _Figura 16. PM2 instalado de forma global._

### 6.5 Clonado de la aplicación

```bash
mkdir ~/apps && cd ~/apps
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app1_3001
git clone https://github.com/marceloquispeortega/api-restful-crud-movies app2_3002
cd ~/apps/app1_3001 && npm install
cd ~/apps/app2_3002 && npm install
```

> 📸 **CAPTURA 17:** Contenido de `~/apps` (`ls -l ~/apps`) y finalización de `npm install` en alguna de las apps.
>
> ![Clonado e instalación](img/17-clone-npm-install.png)
> _Figura 17. Repositorio clonado e instalación de dependencias._

### 6.6 Variables de entorno

```bash
cd ~/apps/app1_3001 && cp .env.example .env
cd ~/apps/app2_3002 && cp .env.example .env
```

**App 1** (`~/apps/app1_3001/.env`):

```bash
PORT=3001
DB_HOST=192.168.10.4
DB_USER=usr_movies
DB_PASSWORD=secret
DB_NAME=db_movies
```

**App 2** (`~/apps/app2_3002/.env`):

```bash
PORT=3002
DB_HOST=192.168.10.4
DB_USER=usr_movies
DB_PASSWORD=secret
DB_NAME=db_movies
```

> 📸 **CAPTURA 18:** Archivos `.env` de ambas apps (`cat .env`). _(Oculta o difumina la contraseña si es necesario.)_
>
> ![Archivos .env](img/18-env-files.png)
> _Figura 18. Variables de entorno de App 1 (puerto 3001) y App 2 (puerto 3002)._

---

## 7. Ejercicio 3: Servidor de base de datos (VM `Lab4.1-DB`)

### 7.1 Hostname (si la VM fue clonada)

```bash
sudo hostnamectl set-hostname db
```

### 7.2 Configuración de IP estática

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: no
      optional: true
      addresses:
        - 192.168.10.4/29
      routes:
        - to: default
          via: 192.168.10.2
      nameservers:
        addresses:
          - 8.8.8.8
```

```bash
sudo netplan apply
```

> 📸 **CAPTURA 19:** Salida de `ip addr` en la VM DB con la IP `192.168.10.4/29`.
>
> ![ip addr DB](img/19-ip-addr-db.png)
> _Figura 19. Configuración de red de `Lab4.1-DB`._

### 7.3 Instalación de MariaDB

```bash
sudo apt install mariadb-server -y
```

> 📸 **CAPTURA 20:** Estado del servicio MariaDB (`systemctl status mariadb`).
>
> ![Estado MariaDB](img/20-mariadb-status.png)
> _Figura 20. MariaDB instalado y en ejecución._

### 7.4 Hardening del servicio

```bash
sudo mysql_secure_installation
```

Respuestas aplicadas durante el asistente:

| Pregunta | Respuesta |
| - | - |
| Switch to unix_socket authentication | _(Y/n)_ |
| Change the root password | _(Y/n)_ |
| Remove anonymous users | Y |
| Disallow root login remotely | Y |
| Remove test database and access to it | Y |
| Reload privilege tables now | Y |

> 📸 **CAPTURA 21:** Ejecución de `mysql_secure_installation` (pasos principales).
>
> ![mysql_secure_installation](img/21-mysql-secure-installation.png)
> _Figura 21. Asistente de aseguramiento de MariaDB._

### 7.5 Configuración de acceso por red

Se modificó `/etc/mysql/mariadb.conf.d/50-server.cnf`:

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

```ini
bind-address = 192.168.10.4
```

```bash
sudo systemctl restart mariadb
```

> 📸 **CAPTURA 22:** Línea `bind-address = 192.168.10.4` en el archivo de configuración y verificación de que escucha en la IP interna (`ss -tlnp | grep 3306`).
>
> ![bind-address](img/22-bind-address.png)
> _Figura 22. MariaDB escuchando en la IP interna._

### 7.6 Creación de base de datos y usuario

```sql
CREATE DATABASE db_movies;
CREATE USER 'usr_movies'@'192.168.10.3' IDENTIFIED BY 'secret';
GRANT ALL PRIVILEGES ON db_movies.* TO 'usr_movies'@'192.168.10.3';
FLUSH PRIVILEGES;
```

> 📸 **CAPTURA 23:** Sesión en el CLI de MariaDB con la creación de la base de datos, el usuario y los privilegios.
>
> ![Creación BD y usuario](img/23-create-db-user.png)
> _Figura 23. Base de datos `db_movies` y usuario `usr_movies` creados._

### 7.7 Creación de la tabla `movies` y carga de datos

Conexión con el nuevo usuario:

```bash
mysql -u usr_movies -h 192.168.10.4 -p
```

```sql
USE db_movies;

CREATE TABLE movies (
    id serial PRIMARY KEY,
    title character varying(150) NOT NULL,
    year integer,
    UNIQUE(title)
);

INSERT INTO movies (title, year) VALUES
    ('Inception', 2010),
    ('The Matrix', 1999),
    ('Pulp Fiction', 1994),
    ('The Dark Knight', 2008),
    ('Eternal Sunshine of the Spotless Mind', 2004),
    ('Forrest Gump', 1994),
    ('Fight Club', 1999),
    ('The Godfather', 1972),
    ('Interstellar', 2014),
    ('Parasite', 2019);
```

> 📸 **CAPTURA 24:** Conexión con `usr_movies`, ejecución de `CREATE TABLE` y `INSERT`.
>
> ![Creación tabla movies](img/24-create-table-movies.png)
> _Figura 24. Creación de la tabla `movies` y carga de registros._

> 📸 **CAPTURA 25:** Resultado de `SELECT * FROM movies;` mostrando los 10 registros.
>
> ![SELECT movies](img/25-select-movies.png)
> _Figura 25. Contenido de la tabla `movies`._

---

## 8. Ejercicio 4: Lanzamiento de instancias con PM2 (VM `Lab4.1-Apps`)

### 8.1 Prueba manual de las aplicaciones

```bash
cd ~/apps/app1_3001 && node app.js
```

Salida esperada:

```
Servidor ejecutándose en el puerto 3001
Conexión a MariaDB exitosa. Pool creado y probado.
```

Se detuvo con `Ctrl+C` y se repitió para la App 2:

```bash
cd ~/apps/app2_3002 && node app.js
```

> 📸 **CAPTURA 26:** Ejecución manual de la App 1 con el mensaje de conexión exitosa a MariaDB.
>
> ![Prueba manual App 1](img/26-node-app1.png)
> _Figura 26. App 1 conectada correctamente a la base de datos._

> 📸 **CAPTURA 27:** Ejecución manual de la App 2 con el mensaje de conexión exitosa a MariaDB.
>
> ![Prueba manual App 2](img/27-node-app2.png)
> _Figura 27. App 2 conectada correctamente a la base de datos._

### 8.2 Lanzamiento con PM2

```bash
cd ~/apps/app1_3001 && pm2 start app.js --name app1_3001
cd ~/apps/app2_3002 && pm2 start app.js --name app2_3002
pm2 status
```

> 📸 **CAPTURA 28:** Salida de `pm2 status` con ambas apps en estado `online`.
>
> ![pm2 status](img/28-pm2-status.png)
> _Figura 28. Procesos `app1_3001` y `app2_3002` en línea bajo PM2._

### 8.3 Auto-arranque

```bash
pm2 startup
pm2 save
```

> 📸 **CAPTURA 29:** Salida de `pm2 startup` y `pm2 save`.
>
> ![pm2 startup y save](img/29-pm2-startup-save.png)
> _Figura 29. Configuración de auto-arranque de PM2._

### 8.4 Verificación del balanceo de carga

Desde la VM Proxy:

```bash
curl http://192.168.10.2
```

Desde la PC anfitriona (mediante el reenvío de puertos):

```bash
curl http://127.0.0.1:80/movies
```

> 📸 **CAPTURA 30:** Varias ejecuciones consecutivas de `curl http://192.168.10.2` desde la VM Proxy mostrando respuestas **alternadas** entre App 1 y App 2.
>
> ![Balanceo desde Proxy](img/30-balanceo-curl-proxy.png)
> _Figura 30. Respuestas alternadas entre App 1 y App 2._

> 📸 **CAPTURA 31:** Resultado de `curl http://127.0.0.1:80/movies` desde la PC anfitriona devolviendo el listado de películas en JSON.
>
> ![Endpoint /movies](img/31-curl-movies-host.png)
> _Figura 31. Endpoint `/movies` accedido desde la PC anfitriona._

> 📸 **CAPTURA 32 (opcional):** Acceso a `http://127.0.0.1` desde el navegador de la PC anfitriona.
>
> ![Acceso desde navegador](img/32-navegador-proxy.png)
> _Figura 32. Acceso web al proxy desde el navegador._

---

## 9. Ejercicio 5: Servicios de monitoreo (VM `Lab4.1-Proxy`)

### 9.1 Instalación de Grafana

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install grafana
sudo systemctl daemon-reload
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

> 📸 **CAPTURA 33:** Estado del servicio Grafana (`systemctl status grafana-server`).
>
> ![Estado Grafana](img/33-grafana-status.png)
> _Figura 33. Grafana instalado y en ejecución._

### 9.2 Instalación de Prometheus y Node Exporter (Proxy)

```bash
sudo apt install prometheus prometheus-node-exporter -y
curl http://localhost:9100/metrics
```

> 📸 **CAPTURA 34:** Salida parcial de `curl http://localhost:9100/metrics` en la VM Proxy.
>
> ![Métricas Node Exporter Proxy](img/34-metrics-proxy.png)
> _Figura 34. Métricas de Node Exporter en `Lab4.1-Proxy`._

### 9.3 Node Exporter en `Lab4.1-Apps` y `Lab4.1-DB`

```bash
sudo apt install prometheus-node-exporter -y
curl http://localhost:9100/metrics
```

> 📸 **CAPTURA 35:** Métricas de Node Exporter en la VM Apps.
>
> ![Métricas Node Exporter Apps](img/35-metrics-apps.png)
> _Figura 35. Métricas de Node Exporter en `Lab4.1-Apps`._

> 📸 **CAPTURA 36:** Métricas de Node Exporter en la VM DB.
>
> ![Métricas Node Exporter DB](img/36-metrics-db.png)
> _Figura 36. Métricas de Node Exporter en `Lab4.1-DB`._

### 9.4 Configuración de Prometheus

Se editó `/etc/prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-proxy'
    static_configs:
      - targets: ['192.168.10.2:9100']

  - job_name: 'node-apps'
    static_configs:
      - targets: ['192.168.10.3:9100']

  - job_name: 'node-db'
    static_configs:
      - targets: ['192.168.10.4:9100']
```

```bash
sudo systemctl restart prometheus
sudo systemctl status prometheus
```

> 📸 **CAPTURA 37:** Sección `scrape_configs` del archivo `prometheus.yml`.
>
> ![prometheus.yml](img/37-prometheus-yml.png)
> _Figura 37. Targets configurados en Prometheus._

> 📸 **CAPTURA 38:** Estado del servicio Prometheus (`systemctl status prometheus`).
>
> ![Estado Prometheus](img/38-prometheus-status.png)
> _Figura 38. Prometheus activo tras aplicar la configuración._

> 📸 **CAPTURA 39 (recomendada):** Interfaz de Prometheus → **Status → Targets** con los 4 targets en estado `UP`.
>
> ![Targets Prometheus](img/39-prometheus-targets.png)
> _Figura 39. Targets de Prometheus en estado UP._

### 9.5 Integración y visualización en Grafana

Pasos realizados:

1. Acceso a Grafana desde la PC anfitriona: `http://127.0.0.1:8080` (usuario `admin`, cambio de contraseña en el primer ingreso).
2. Adición de la fuente de datos **Prometheus** con URL `http://localhost:9090`.
3. Importación de los dashboards **Node Exporter Full** (ID `1860`) y **Node Exporter Full with Node Name** (ID `10242`).

> 📸 **CAPTURA 40:** Pantalla de inicio de sesión de Grafana accedida desde `http://127.0.0.1:8080`.
>
> ![Login Grafana](img/40-grafana-login.png)
> _Figura 40. Acceso a Grafana mediante reenvío de puertos._

> 📸 **CAPTURA 41:** Configuración del Data Source Prometheus con el mensaje _"Successfully queried the Prometheus API"_.
>
> ![Data source Prometheus](img/41-grafana-datasource.png)
> _Figura 41. Fuente de datos Prometheus en Grafana._

> 📸 **CAPTURA 42:** Importación de los dashboards por ID (1860 y 10242).
>
> ![Importar dashboards](img/42-grafana-import.png)
> _Figura 42. Importación de dashboards de Node Exporter._

> 📸 **CAPTURA 43:** Dashboard con métricas de la VM **Proxy** (selector _Host/Instance_ en `192.168.10.2:9100`).
>
> ![Dashboard Proxy](img/43-dashboard-proxy.png)
> _Figura 43. Métricas de `Lab4.1-Proxy` en Grafana._

> 📸 **CAPTURA 44:** Dashboard con métricas de la VM **Apps** (`192.168.10.3:9100`).
>
> ![Dashboard Apps](img/44-dashboard-apps.png)
> _Figura 44. Métricas de `Lab4.1-Apps` en Grafana._

> 📸 **CAPTURA 45:** Dashboard con métricas de la VM **DB** (`192.168.10.4:9100`).
>
> ![Dashboard DB](img/45-dashboard-db.png)
> _Figura 45. Métricas de `Lab4.1-DB` en Grafana._

---

## 10. Pruebas de alta disponibilidad (failover)

Se simuló la caída de una instancia de la aplicación para verificar que el balanceador redirige el tráfico a la instancia disponible.

```bash
# En la VM Apps: detener la App 1
pm2 stop app1_3001
pm2 status

# Desde Proxy o PC anfitriona: repetir peticiones
curl http://192.168.10.2/movies
```

> 📸 **CAPTURA 46:** `pm2 status` con `app1_3001` en estado `stopped` y `app2_3002` en `online`.
>
> ![App 1 detenida](img/46-pm2-app1-stopped.png)
> _Figura 46. Simulación de caída de la App 1._

> 📸 **CAPTURA 47:** Peticiones `curl` consecutivas respondiendo **únicamente desde App 2**, sin errores.
>
> ![Failover curl](img/47-failover-curl.png)
> _Figura 47. El balanceador redirige todo el tráfico a la App 2._

Se restableció el servicio:

```bash
pm2 start app1_3001
```

> 📸 **CAPTURA 48:** Tras reiniciar la App 1, las respuestas vuelven a alternarse entre ambas instancias.
>
> ![Recuperación balanceo](img/48-recuperacion.png)
> _Figura 48. Balanceo restablecido con ambas instancias._

> 📸 **CAPTURA 49 (opcional):** Gráfico de Grafana mostrando la variación de CPU/red de la VM Apps durante la prueba.
>
> ![Grafana durante failover](img/49-grafana-failover.png)
> _Figura 49. Impacto de la prueba de failover en las métricas._

---

## 11. Problemas encontrados y soluciones

_(Completa esta tabla con los inconvenientes reales que tuviste. Si no hubo ninguno, indícalo.)_

| N.º | Problema | Causa | Solución aplicada |
| - | - | - | - |
| 1 | _(ej. Apps sin salida a Internet)_ | _(ej. falta `ip_forward` o regla MASQUERADE)_ | _(…)_ |
| 2 | _(…)_ | _(…)_ | _(…)_ |
| 3 | _(…)_ | _(…)_ | _(…)_ |

---

## 12. Conclusiones y reflexiones

_(Redacta con tus propias palabras. Puedes guiarte por estas preguntas:)_

1. **Sobre la alta disponibilidad:** ¿Qué ocurrió cuando una de las apps dejó de funcionar? ¿Qué ventaja aporta el balanceador frente a una sola instancia?
2. **Sobre la segregación de servicios:** ¿Por qué es conveniente separar proxy, aplicaciones y base de datos en VMs distintas? ¿Qué riesgos de seguridad se reducen?
3. **Sobre el acceso a la base de datos:** ¿Qué aporta restringir el usuario `usr_movies` a la IP `192.168.10.3` y no exponer `root` de forma remota?
4. **Sobre PM2:** ¿Qué beneficios ofrece frente a ejecutar `node app.js` manualmente?
5. **Sobre el monitoreo:** ¿Qué métricas fueron más útiles para entender el estado de cada VM?
6. **Limitaciones de la arquitectura:** ¿Qué puntos únicos de falla siguen existiendo (por ejemplo, el proxy, la base de datos o que ambas apps estén en la misma VM)? ¿Cómo se podrían mitigar?
7. **Aprendizajes y mejoras:** ¿Qué fue lo más difícil y qué mejorarías?

**Conclusión general:**

> _(Escribe aquí un párrafo de cierre.)_

---

## 13. Anexos

### Anexo A. Resumen de comandos de verificación

| Verificación | Comando | VM |
| - | - | - |
| Direccionamiento IP | `ip addr` | Todas |
| Validar Nginx | `sudo nginx -t` | Proxy |
| Reglas NAT | `sudo iptables -t nat -L -n -v` | Proxy |
| Estado de apps | `pm2 status` | Apps |
| Acceso a la BD | `mysql -u usr_movies -h 192.168.10.4 -p` | Apps / DB |
| Balanceo | `curl http://192.168.10.2` | Proxy |
| Métricas del SO | `curl http://localhost:9100/metrics` | Todas |
| Estado de servicios | `systemctl status prometheus grafana-server` | Proxy |

### Anexo B. Lista de verificación de capturas requeridas por la rúbrica

- [ ] Configuración de red en cada VM (`ip addr`) — Capturas 07, 13, 19
- [ ] Configuración de Nginx y resultado de `sudo nginx -t` — Capturas 09, 10
- [ ] Estado de las aplicaciones con `pm2 status` — Captura 28
- [ ] Conexión a la base de datos y creación de la tabla `movies` — Capturas 23 a 25
- [ ] Pruebas de balanceo de carga (respuestas alternadas App 1 / App 2) — Captura 30
- [ ] Grafana mostrando métricas de las tres VMs — Capturas 43, 44, 45
- [ ] Conclusiones y reflexiones — Sección 12

### Anexo C. Referencias

- Documentación oficial de Nginx: <https://nginx.org/en/docs/>
- Node.js: <https://nodejs.org/es/download>
- PM2: <https://pm2.keymetrics.io/docs/usage/quick-start/>
- MariaDB: <https://mariadb.com/kb/en/documentation/>
- Prometheus: <https://prometheus.io/docs/>
- Grafana: <https://grafana.com/docs/>
- Repositorio de la aplicación: <https://github.com/marceloquispeortega/api-restful-crud-movies>
