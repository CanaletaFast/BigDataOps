# 🚀 CONFIGURACIÓN DOCKER (MEJORADA)

### ⚙️ Cambiado el puerto de Jupyter en el archivo `docker-compose.yaml`:
-  Así evitamos conflictos con **Hue**.
```yaml
jupyter:
  platform: linux/amd64
  image: base-jupyter:latest
  container_name: jupyter
  ports:
    - 8889:8888  # Cambiado a 8889 para evitar el conflicto
    # - 8888:8888
    - 4040:4040
  environment:
    - JUPYTER_ENABLE_LAB=yes
  depends_on:
    - spark-master
  volumes:
    - ./jupyter/notebooks:/home/jovyan/notebooks
    - ./jupyter/data:/home/jovyan/data
```

---

## 🔑 Cambios Esenciales en el Sistema Docker para la Configuración Exitosa de Hive y PostgreSQL (Guardar de HDFS a local)

### 1. 🛠 Configuración del fichero `entrypoint.sh` para HDFS (`core-site.xml`):
-  Se añadió la configuración de `fs.defaultFS` en el archivo `entrypoint.sh` de **Hadoop**, para asegurarnos de que HDFS escriba y lea desde `namenode` correctamente:
```bash
addProperty /etc/hadoop/core-site.xml fs.defaultFS hdfs://namenode:9000 # Añadido, permite escritura HDFS
```

### 2. 📝 Configuración del Archivo `hive-site.xml` en el Contenedor `hiveserver2`:
-  Este archivo `hive-site.xml` contiene configuraciones específicas de **Hive** y fue reemplazado en el contenedor `hiveserver2` para asegurarnos de que el directorio de almacenamiento (warehouse) sea accesible en HDFS, evitando problemas de permisos y de lectura/escritura. 
```bash
docker cp hive-site.xml hiveserver2:/opt/hive/conf/hive-site.xml
```
-  La configuración importante dentro de `hive-site.xml` (`hive` ➔ `hive-site.xml`) que evitó errores de permisos en warehouse fue reemplazar: 
```xml
<!-- <property>
        <name>metastore.warehouse.dir</name>
        <value>/opt/hive/data/warehouse</value>
    </property> -->
    <property> <!--He añadido esto para evitar problemas de permisos-->
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
```

### 3. 👤 Configuración del `docker-compose.yml` para `hiveserver2` como root:
-  La ejecución del contenedor `hiveserver2` como **root** permitió realizar cambios de permisos en los directorios dentro de los contenedores, lo que fue esencial para que el directorio warehouse fuera accesible.
-  Se agregó la línea `user: root` en el contenedor `hiveserver2` dentro de `docker-compose.yml`:
```yaml
hiveserver2:
    image: apache/hive:4.0.0
    depends_on:
      - metastore
    restart: unless-stopped
    container_name: hiveserver2
    environment:
      SERVICE_OPTS: "-Dhive:metastore:uris=thrift://metastore:9083"
      VERBOSE: "true"
      SERVICE_NAME: 'hiveserver2'
    ports:
      - '10000:10000'
      - '10002:10002'
    volumes:
      - ./hive/warehouse:/opt/hive/data/warehouse
    user: root  # Añade esta línea para ejecutar como root
```

### 4. 📥 Añadir el Driver de PostgreSQL para Conexión al Metastore:
-  **Hive** necesita el driver `org.postgresql.Driver` para comunicarse con el metastore en **PostgreSQL**.
-  Se movió el archivo `postgresql-42.7.4.jar` al directorio `/opt/hive/lib/` en el contenedor `hiveserver2`:
```bash
docker cp ./postgresql-42.7.4.jar hiveserver2:/opt/hive/lib/
```
-  Esto permitió que **Hive** pueda conectar con **PostgreSQL** para el metastore, habilitando la persistencia de los metadatos en PostgreSQL y evitando el error `ClassNotFoundException : org.postgresql.Driver`.

### 5. 🔐 Asignación de Permisos en el Directorio warehouse dentro de `hiveserver2`:
-  Para que los procesos de **Hive** puedan leer y escribir en el directorio warehouse, fue necesario asignar permisos amplios (777) a ese directorio dentro del contenedor:
```bash
docker exec -it hiveserver2 bash -c "chmod -R 777 /opt/hive/data/warehouse"
```

---