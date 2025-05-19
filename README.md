# ⚙️ Guía paso a paso para reiniciar y levantar un clúster de Spark

- [Paso 1: Eliminar todo](#paso-1-eliminar-todo)
- [Paso 2: Asignar IP manual](#paso-2-asignar-ip-manual-y-preparar-red)
- [Paso 3: Levantar DHCP](#paso-3-levantar-servidor-dhcp)
- [Paso 4: Inicializar Swarm](#paso-4-inicializar-docker-swarm)
- [Paso 5: Desplegar Spark y NFS](#paso-5-desplegar-clúster-de-spark)
- [Paso 6: Probar con PySpark](#paso-6-probar-el-clúster-con-código-pyspark)
- [Cierre de sesión](#cierre-seguro-de-sesión-fin-de-jornada)


## Datos de prueba

https://drive.google.com/file/d/1e4Z0L3aoBff9HwYb2v8EMuOGF82MqQ5p/view

## ⚠️ Salir de cualquier Swarm previo (Master y Nodos)

Antes de comenzar, asegúrate de salir de cualquier swarm existente:

```bash
docker swarm leave --force
```

---

## ✅ Paso 1: Eliminar todo

### 🧹 Eliminar el stack de Spark actual (Master)

```bash
docker stack rm sparkcluster
```

### 🧽 Limpieza total de Docker (contenedores, imágenes, volúmenes, etc.) (Master y Nodos)

```bash
docker system prune -a --volumes
```

### 🔄 Eliminar IP Manual (Master)

Si te asignaste una IP manualmente al switch, deberas eliminarla para que el DHCP te asigne una a continuación:

```bash
nmcli connection show

sudo nmcli connection modify "Wired connection 1" \
  ipv4.method auto \
  ipv4.addresses "" \
  ipv4.gateway "" \
  ipv4.dns ""

sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up   "Wired connection 1"
```

---

## ✅ Paso 2: Asignar IP manual y preparar red

### 🛠️ Asignar IP manual al nodo principal (Master)

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.addresses 192.168.0.1/24 \
  ipv4.gateway   192.168.0.1 \
  ipv4.dns       "8.8.8.8 1.1.1.1" \
  ipv4.method    manual

sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up   "Wired connection 1"
```

### 📡 Verificar la IP asignada (Master)

```bash
ping -c 3 192.168.0.1  # Se hace ping a sí mismo
```

---

## ✅ Paso 3: Levantar servidor DHCP

### 🚀 Levantar DHCP con Docker Compose (Master)

```bash
docker compose -f dhcp-compose.yml up -d
```

### 👀 Ver registros de conexión (Master)

Así se puede ver a quién se le van asignando las IPs:

```bash
docker logs dhcp -f
```

### 🛑 (Opcional) Detener el DHCP una vez que todos estén conectados (Master)

```bash
docker compose -f dhcp-compose.yml down
```

---

## ✅ Paso 4: Inicializar Docker Swarm

### 🐳 Inicializar swarm desde el nodo principal (Master)

```bash
docker swarm init --advertise-addr 192.168.0.1
```

### 📋 Ejemplo de salida de swarm init

```bash
Swarm initialized: current node (...) is now a manager.
To add a worker: docker swarm join --token ... 192.168.0.1:2377
```

### 🔗 Paso 4.1: Agregar nodos workers al Swarm (Nodos)

En cada nodo worker, ejecuta el comando proporcionado por el nodo manager:

```bash
docker swarm join --token <TOKEN_DEL_WORKER> 192.168.0.1:2377
```

Si perdiste el token, puedes recuperarlo en el manager con:

```bash
docker swarm join-token worker
```

---

## ✅ Paso 5: Desplegar clúster de Spark

### 🗄️ Paso 5.1: Usar volumen compartido vía NFS

#### Paso 1: Preparar el servidor NFS (solo Master)

```bash
sudo apt update
sudo apt install nfs-kernel-server -y

sudo mkdir -p /srv/nfs/datos_jupyter
sudo chmod -R 777 /srv/nfs/datos_jupyter
```

#### Paso 2: Configurar exportaciones NFS (solo Master)

```bash
sudo nano /etc/exports
```

Agrega:

```
/srv/nfs/datos_jupyter 192.168.0.0/24(rw,sync,no_subtree_check,no_root_squash)
```

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

#### Paso 3: Montar NFS en todos los nodos (Master y Nodos)

```bash
sudo apt install nfs-common -y

sudo mkdir -p /mnt/datos_jupyter
sudo mount -t nfs 192.168.0.1:/srv/nfs/datos_jupyter /mnt/datos_jupyter
```

#### Paso 4: Copiar tus archivos CSV a la carpeta compartida (solo Master)

```bash
cp ./ibm_card_txn/*.csv /srv/nfs/datos_jupyter/
```

> Asegúrate de que `/srv/nfs/datos_jupyter/` es accesible desde todos los nodos.

#### Paso 5: En tu `docker-compose.yml`, declara el volumen así:

```yaml
volumes:
  datos_jupyter:
    driver: local
    driver_opts:
      type: "nfs"
      o: "addr=192.168.0.1,nolock,soft,rw"
      device: ":/srv/nfs/datos_jupyter"
```
### Paso 5.2: Asignar el rol a cada nodo y modificar el compose

### Paso 5.2.1: Asignar tipo a cada nodo:

```bash
# Ejemplo
docker node update --label-add tipo=[TIPO] [NombreDelHostname]
# El nombre de los hostname se puede sacar con docker node ls
```

```bash
docker node update --label-add tipo=high ubuntu-ASUS-TUF-Gaming-F15-FX506HF-FX506HF
docker node update --label-add tipo=high roberto-baeza
docker node update --label-add tipo=high bruno
docker node update --label-add tipo=high yago
```

# Paso 5.2.2: Modificar el compose

En el archivo `spark-compose.yml` en cada uno de los tipos de worker (`spark-worker-low`, `spark-worker-medium`, `spark-worker-high`) se debe adaptar el numero de replicas que se deben asignar

### 🚀 Paso 5.3: Desplegar el stack de Spark (solo Master)

```bash
docker stack deploy -c spark-compose.yml sparkcluster
```

### 🌐 Interfaces disponibles (accesibles desde Master)

```
Jupyter: http://127.0.0.1:8888/login?next=%2Flab%3F
# Token: miclustertoken
Spark UI: http://0.0.0.0:8080/
```

---

## ✅ Paso 6: Probar el clúster con código PySpark

### 🧪 Código Python de prueba con PySpark

```python
from pyspark.sql import SparkSession

# Crea la sesión de Spark (ya configurada en tu contenedor con master)
spark = SparkSession.builder.appName("HolaMundoCluster").getOrCreate()

# Lee todos los CSV del volumen montado
df = spark.read.option("header", True).csv("/data/*.csv")

# Muestra el esquema inferido por Spark
df.printSchema()

# Muestra los primeros 5 registros
df.show(5)

# Cuenta total de registros (acción distribuida)
print(f"Total de filas: {df.count()}")

# Selecciona una columna si existe, por ejemplo "txn_amount"
if "txn_amount" in df.columns:
    df.select("txn_amount").show(5)

# Operación simple: contar cuántas filas tiene cada archivo si tienen columna identificadora
if "txn_id" in df.columns:
    df.groupBy("txn_id").count().show()
```

### 📋 Logs y monitoreo (solo Master)

#### Para observar al master

```bash
docker service logs sparkcluster_spark-master -f

docker service ps sparkcluster_spark-master
```

#### Para observar jupyter

```bash
docker service logs sparkcluster_jupyter -f

docker service ps sparkcluster_jupyter
```

#### Estado de ambos servicios

```bash
docker service ps sparkcluster_spark-master sparkcluster_jupyter
```

---

## ✅ Cierre seguro de sesión (fin de jornada)

Si vas a apagar tu entorno o detener todo por hoy, sigue estos pasos:

### 🔻 1. Apagar el stack de Spark (solo Master)

```bash
docker stack rm sparkcluster
```

### 🔻 2. Desmontar NFS en todos los nodos (Master y Nodos)

```bash
sudo umount /mnt/datos_jupyter
```

### 🔻 3. (Opcional) Apagar el servidor NFS (solo Master)

```bash
sudo systemctl stop nfs-kernel-server
```

### 🔻 4. (Opcional) Salir del Swarm (todos los nodos)

```bash
docker swarm leave --force
```

### 🔻 5. (Opcional) Apagar el sistema (Master y Nodos)

```bash
sudo poweroff
```

---

## 🛠️ Problema común: Ethernet bloquea el uso de Wi-Fi

### ❗ Síntoma

En algunas computadoras, al conectar el cable Ethernet, la red Wi-Fi deja de tener acceso a internet o deja de funcionar correctamente.

Esto ocurre porque el sistema establece la ruta por defecto (`default route`) a través del dispositivo Ethernet, priorizándolo sobre el Wi-Fi.

### ✅ Solución

1. **Identifica el nombre de la interfaz de red cableada.**
   Puedes usar el siguiente comando para listar todas las interfaces:

   ```bash
   ip addr
   ```

   Busca un nombre que inicie con `enx` o `eth` (ejemplo: `enx5c531029014b`).

2. **Elimina la ruta por defecto de esa interfaz:**

   ```bash
   sudo ip route del default dev <nombre_de_la_interfaz>
   ```

   Por ejemplo:

   ```bash
   sudo ip route del default dev enx5c531029014b
   ```

3. **Verifica que la red Wi-Fi vuelva a tener prioridad:**

   ```bash
   ip route
   ```

   Deberías ver que la interfaz `wlan0` (o similar) ahora tiene la ruta por defecto.
