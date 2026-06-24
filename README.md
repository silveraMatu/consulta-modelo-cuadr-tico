# 🧮 Sistema Cuadrático Distribuido: Frontend, Backend y Caché

**Autor:** Silvera Matías

Este proyecto implementa una arquitectura de microservicios distribuida en tres máquinas virtuales (VMs) independientes sobre una red NAT. El objetivo del sistema es resolver la ecuación matemática cuadrática $y = 2x^2 + 5x + 3$ optimizando los tiempos de respuesta mediante un patrón *Cache-Aside*.

---

## 🏛️ Arquitectura y Flujo de Datos

El entorno consta de tres servidores Linux interconectados:
1.  **VM-Frontend (10.0.2.10):** Servidor web web estático proxy inverso (Nginx).
2.  **VM-Backend (10.0.2.20):** API y lógica de negocio (Node.js + Express).
3.  **VM-Cache (10.0.2.30):** Base de datos en memoria (Redis).

**Flujo de la petición (Request / Response):**
* El cliente ingresa un valor $X$ en el navegador.
* El **Frontend** envía la petición HTTP al **Backend**.
* El **Backend** verifica en la **Caché (Redis)** si ese valor $X$ ya fue calculado.
    * 🟢 **Hit (Encontrado):** Redis devuelve el resultado inmediatamente.
    * 🔴 **Miss (No encontrado):** El Backend calcula la ecuación, guarda el resultado en Redis limitando el historial a las últimas 5 consultas, y devuelve la respuesta.

---

## ⚙️ Paso a Paso del Despliegue

### 1. Preparación de las Máquinas (Clonación y Red)
Se utilizó una VM base limpia que fue clonada tres veces ("Clonación completa").
* **Crucial:** Se generaron nuevas direcciones MAC para cada adaptador de red durante la clonación para evitar colisiones de IP.
* Se configuraron túneles de **Reenvío de Puertos** en VirtualBox para gestionar las máquinas vía SSH desde el host anfitrión:
    * Frontend: `Puerto 2210` $\rightarrow$ `10.0.2.10:22`
    * Backend: `Puerto 2220` $\rightarrow$ `10.0.2.20:22`
    * Caché: `Puerto 2230` $\rightarrow$ `10.0.2.30:22`

### 2. VM-Cache (Configuración de Redis)
Se instaló el servidor de base de datos en memoria para gestionar el historial.
```bash
sudo apt update && sudo apt install redis-server -y
sudo hostnamectl set-hostname cache

Se configuró una IP estática editando el archivo de Netplan (/etc/netplan/50-cloud-init.yaml) estableciendo la IP en 10.0.2.30.

3. VM-Backend (Configuración de la API Node.js)
Se configuró el entorno de ejecución para el código princip

sudo apt update && sudo apt install nodejs npm -y
sudo hostnamectl set-hostname backend
npm install express redis cors

Se fijó la IP estática en 10.0.2.20 mediante Netplan. El script server.js gestiona la validación, el cálculo matemático y la inserción/eliminación dinámica en Redis para mantener solo los últimos $n$ registros.4. VM-Frontend (Configuración de Nginx)Se desplegó el servidor web para servir la interfaz gráfica al cliente.

sudo apt update && sudo apt install nginx -y
sudo hostnamectl set-hostname frontend

Se modificó /etc/nginx/sites-available/default configurando un Proxy Inverso para que las peticiones a /api/ sean redirigidas internamente al backend (http://10.0.2.20:3000/). La IP estática se fijó en 10.0.2.10.

🛠️ Resolviendo Errores Frecuentes (Troubleshooting)
Durante la fase de desarrollo e integración surgieron algunos desafíos técnicos que fueron diagnosticados y resueltos:

Error: ssh: connect to host 127.0.0.1 port 2230: Connection refused

Causa: La regla de reenvío de puertos de VirtualBox apuntaba a una IP equivocada (10.0.2.15), pero por DHCP la máquina había tomado otra (10.0.2.17).

Solución: Entrar directamente desde la interfaz gráfica, usar el comando ip a para verificar la IP real y actualizar la regla NAT correspondiente.

Error: Backend arrojando ECONNREFUSED 10.0.2.30:6379 en Node.js.

Causa: Redis es restrictivo por defecto. El comando sudo ss -tln | grep 6379 demostró que la caché estaba escuchando exclusivamente en localhost (127.0.0.1). Había múltiples declaraciones de la variable "bind" en el archivo de configuración.

Solución: Editar /etc/redis/redis.conf, eliminar las referencias al localhost y descomentar la línea principal dejándola estrictamente como bind 0.0.0.0. Finalmente, se reinició el servicio con sudo systemctl restart redis-server.

🚀 Ejecución del Proyecto
Para levantar el sistema de cero, sigue este orden:

1. Levantar Caché (VM-Cache):
El servicio de Redis arranca automáticamente con el sistema, pero puedes verificar su estado con:

Bash
sudo systemctl status redis-server

2. Levantar API (VM-Backend):
Ingresa al directorio del proyecto e inicia el servidor de Node:

Bash
cd ~/api-backend
node server.js

# Esperar el mensaje: "Conectado exitosamente a la VM-Cache (Redis)"
3. Visualizar Interfaz (Host / Cliente):
Asegúrate de que la VM-Frontend esté encendida. Hemos expuesto el puerto 80 de esta máquina al puerto 8080 de nuestro equipo anfitrión.
Abre tu navegador web e ingresa a:
👉 http://localhost:8080
