# Ficheros de Configuración Clave

## **Configuración DNS**

#### Archivo: `/etc/bind/named.conf.local`

**Función:**
Define las zonas DNS administradas por el servidor BIND9.

**Configuración implementada:**

```conf
zone "gob.internal" {
    type master;
    file "/etc/bind/db.gob.local";
};

zone "234.139.10.in-addr.arpa" {
    type master;
    file "/etc/bind/db.234.139.10";
};
```

**Descripción:**

* Se configuró la zona directa `gob.internal`.
* Se configuró la zona inversa `234.139.10.in-addr.arpa`.
* Permite resolver nombres de host e IPs dentro de la infraestructura.

---

#### Archivo: `/etc/bind/db.gob.local`

**Función:**
Base de datos de resolución directa.

**Configuración implementada:**

```dns
dns      IN A 10.139.234.201
security IN A 10.139.234.202
proxy    IN A 10.139.234.203
web1     IN A 10.139.234.204
web2     IN A 10.139.234.205
grafana  IN A 10.139.234.206
base     IN A 10.139.234.207

tramites IN A 10.139.234.203
```

**Descripción:**

Permite acceder a los servicios utilizando nombres DNS en lugar de direcciones IP.

**Ejemplo:**

```text
tramites.gob.internal
```

---

#### Archivo: `/etc/bind/db.234.139.10`

**Función:**
Base de datos de resolución inversa.

**Configuración implementada:**

```dns
201 IN PTR dns.gob.internal.
202 IN PTR security.gob.internal.
203 IN PTR proxy.gob.internal.
204 IN PTR web1.gob.internal.
205 IN PTR web2.gob.internal.
206 IN PTR grafana.gob.internal.
207 IN PTR base.gob.internal.
```

**Descripción:**

Permite obtener el nombre asociado a una dirección IP.

**Ejemplo:**

```bash
dig -x 10.139.234.203
```

**Resultado esperado:**

```text
proxy.gob.internal
```

---

### Configuración de Seguridad (Bastion Host)

#### Archivo: `/etc/ssh/sshd_config`

**Función:**
Configuración del servicio OpenSSH.

**Configuración implementada:**

```conf
Port 2222
PermitRootLogin no
PasswordAuthentication yes
```

**Descripción:**

Se modificó el puerto predeterminado SSH (22) por el puerto 2222 para reducir intentos automatizados de acceso y mejorar la seguridad administrativa. Además, se deshabilitó el acceso directo del usuario root.

---

#### Firewall UFW

**Función:**
Control de acceso a los servicios.

**Configuración implementada en Security:**

```bash
sudo ufw allow 2222/tcp
sudo ufw enable
```

**Configuración implementada en DNS:**

```bash
sudo ufw allow from 10.139.234.0/24 to any port 53 proto tcp
sudo ufw allow from 10.139.234.0/24 to any port 53 proto udp
```

**Descripción:**

Se permitió únicamente el tráfico necesario para cada servicio, reduciendo la superficie de ataque y mejorando el control de acceso dentro de la infraestructura.

---

### Scripts de Auditoría

#### Archivo: `/opt/scripts/check_ssh.sh`

**Función:**
Verificar el estado y disponibilidad del servicio SSH.

---

#### Archivo: `/opt/scripts/check_firewall.sh`

**Función:**
Verificar las reglas activas del firewall UFW.

---

#### Archivo: `/opt/scripts/check_users.sh`

**Función:**
Auditar usuarios, grupos y permisos básicos del sistema.

---

#### Archivo: `/opt/scripts/audit_security.sh`

**Función:**
Ejecutar de forma centralizada todas las verificaciones de seguridad y auditoría.

---

**Descripción General:**

Estos scripts permiten automatizar tareas básicas de administración y seguridad, facilitando la supervisión del servidor Security y reduciendo el tiempo necesario para realizar verificaciones manuales.

---

## **Configuración del Proxy NGINX**

### Archivo: `/etc/nginx/sites-available/default`

**Función:**
Implementa el Reverse Proxy y Balanceador de Carga encargado de distribuir las solicitudes entre los servidores WEB1 y WEB2.

**Configuración implementada:**

```nginx
upstream backend {

    server 10.139.234.204:3000;
    server 10.139.234.205:3000;

}

server {

    listen 80;

    server_name tramites.gob.internal;

    location / {

        proxy_pass http://backend;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;

    }

}
```

**Descripción:**

- NGINX recibe todas las solicitudes HTTP de los usuarios.
- Las solicitudes son distribuidas entre WEB1 y WEB2 mediante balanceo de carga.
- Los servidores backend permanecen ocultos para los clientes.
- El acceso principal a la plataforma se realiza mediante:

```text
http://tramites.gob.internal
```

**Prueba realizada:**

```bash
curl http://10.139.234.203
```

**Resultado:**

```text
Respuesta generada por WEB1 o WEB2
```

---
## Configuración de la Aplicación Web (WEB1 y WEB2)

#### Archivo: `~/portal/app.js`

**Función:**

Implementa la aplicación web de registro de trámites desarrollada con Node.js, Express y MariaDB.

**Dependencias instaladas:**

```bash
sudo apt update
sudo apt install nodejs npm -y

mkdir portal
cd portal

npm init -y

npm install express mysql2
```

**Configuración de conexión a la base de datos:**

```javascript
const db = mysql.createConnection({

 host:'10.139.234.207',

 user:'user_paz',

 password:'secret',

 database:'db_tramites'

});
```

**Servicios proporcionados:**

- Formulario de registro de trámites.
- Captura de información del ciudadano.
- Inserción de datos en MariaDB.
- Integración con el balanceador NGINX.

**Puerto utilizado:**

```javascript
app.listen(3000);
```

**Ejecución de la aplicación:**

```bash
node app.js
```

**Verificación del servicio:**

```bash
ss -tulpn | grep 3000
```

**Prueba local:**

```bash
curl http://localhost:3000
```

**Resultado:**

```text
Portal de Trámites Municipales
```

---
