# Requisitos 

Para Windows: 
-Instalar Docker Desktop 
-Tener Docker Compose 
-Asegurarse de que WSL 2 está instalado 

 

Para verificar podemos usar los siguientes comandos: 
```bash
docker –version 
docker-compose –version 
```
 

## Docker + Hadoop + YARN 

1. Crear un nuevo directorio para el proyecto 
2. Crear el archivo docker-compose.yml: 
```bash
services:
  namenode:
    image: bde2020/hadoop-namenode:2.0.0-hadoop3.2.1-java8
    container_name: namenode
    restart: always
    ports:
      - 9870:9870
      - 9000:9000
    volumes:
      - hadoop_namenode:/hadoop/dfs/name
    environment:
      - CLUSTER_NAME=test
    env_file:
      - ./hadoop.env

  datanode:
    image: bde2020/hadoop-datanode:2.0.0-hadoop3.2.1-java8
    container_name: datanode
    restart: always
    ports:
      - 9864:9864
    volumes:
      - hadoop_datanode:/hadoop/dfs/data
    environment:
      SERVICE_PRECONDITION: "namenode:9870"
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode

  resourcemanager:
    image: bde2020/hadoop-resourcemanager:2.0.0-hadoop3.2.1-java8
    container_name: resourcemanager
    restart: always
    ports:
      - 8088:8088
    environment:
      SERVICE_PRECONDITION: "namenode:9000 namenode:9870 datanode:9864"
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode
      - datanode

  nodemanager:
    image: bde2020/hadoop-nodemanager:2.0.0-hadoop3.2.1-java8
    container_name: nodemanager
    restart: always
    ports:
      - 8042:8042
    environment:
      SERVICE_PRECONDITION: "namenode:9000 namenode:9870 datanode:9864 resourcemanager:8088"
    env_file:
      - ./hadoop.env
    depends_on:
      - namenode
      - datanode
      - resourcemanager

volumes:
  hadoop_namenode:
  hadoop_datanode:
```
3. Crear el archivo de variables de entorno hadoop.env: 
```bash
CORE_CONF_fs_defaultFS=hdfs://namenode:9000
CORE_CONF_hadoop_http_staticuser_user=root
CORE_CONF_hadoop_proxyuser_hue_hosts=*
CORE_CONF_hadoop_proxyuser_hue_groups=*
CORE_CONF_io_compression_codecs=org.apache.hadoop.io.compress.SnappyCodec

HDFS_CONF_dfs_webhdfs_enabled=true
HDFS_CONF_dfs_permissions_enabled=false
HDFS_CONF_dfs_namenode_datanode_registration_ip___hostname___check=false

YARN_CONF_yarn_log___aggregation___enable=true
YARN_CONF_yarn_log_server_url=http://historyserver:8188/applicationhistory/logs/
YARN_CONF_yarn_resourcemanager_recovery_enabled=true
YARN_CONF_yarn_resourcemanager_store_class=org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore
YARN_CONF_yarn_resourcemanager_scheduler_class=org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler
YARN_CONF_yarn_scheduler_capacity_root_default_maximum___allocation___mb=8192
YARN_CONF_yarn_scheduler_capacity_root_default_maximum___allocation___vcores=4
YARN_CONF_yarn_resourcemanager_fs_state___store_uri=/rmstate
YARN_CONF_yarn_resourcemanager_system___metrics___publisher_enabled=true
YARN_CONF_yarn_resourcemanager_hostname=resourcemanager
YARN_CONF_yarn_resourcemanager_address=resourcemanager:8032
YARN_CONF_yarn_resourcemanager_scheduler_address=resourcemanager:8030
YARN_CONF_yarn_resourcemanager_resource__tracker_address=resourcemanager:8031
YARN_CONF_yarn_timeline___service_enabled=true
YARN_CONF_yarn_timeline___service_generic___application___history_enabled=true
YARN_CONF_yarn_nodemanager_remote___app___log___dir=/app-logs
```
4. Iniciar el cluster 
```bash
docker compose up –d  
 ```
![img0](https://tajamar365-my.sharepoint.com/:i:/p/tsuenkit_lui/EYlcf0_wTuhFvL6sdmuCt9MBZJvwy3M7M9N_SurYFw-4YA?e=HVlajd)

**Si existe ya un contenedor con ese nombre deberemos borrarlo: 
Buscamos el contenedor con el nombre que ya existe y nos da conflicto 
```bash
docker ps –a 
docker stop CONTAINER_name 
docker rm CONTAINER_name 
```

5. Prueba del cluster 

-Entramos al contenedor namenode 
```bash
docker compose exec namenode bash 
```
Nos debe aparecer un codigo:  
![img1](https://tajamar365-my.sharepoint.com/:i:/p/tsuenkit_lui/ERlL6xRwKAhLtEZi4YotPYUBQDyLoPxwsegGBuye6GflBw?e=ZviUtO)

Crear un directorio de prueba en HDFS 
```bash
hdfs dfs -mkdir -p /user/root/prueba 
```

Crear un archivo de prueba 
```bash
echo "Hola Mundo estoy testeando" > prueba.txt 
```
Podemos comprobar que esta creado con un: 
```bash
ls 
```

Cargamos el archivo desde el sistema de archivos local a HDFS 
```bash
hdfs dfs -put prueba.txt /user/root/prueba 
```

Verificar que el archivo está en HDFS 
```bash
hdfs dfs -cat /user/root/prueba/prueba.txt 
```

 ## Ejemplo WordCount 
 

1. Crear directorio para input 
```bash
hdfs dfs -mkdir -p /user/root/wordcount/input 
```
  

2. Crear archivo de prueba 
```bash
echo "Hola mundo Hola world" > input.txt 
```
```bash
hdfs dfs -put input.txt /user/root/wordcount/input 
```
 
3. Comprobamos que se ha cargado el fichero con: 
```bash 
hdfs dfs -ls /user/root/wordcount/input 
```

4. Ejecutar WordCount 
```bash
hadoop jar /opt/hadoop-3.2.1/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.2.1.jar wordcount /user/root/wordcount/input /user/root/wordcount/output 
```
Si ya existe porque lo hemos ejecutado antes podemos borrar el directorio output:
```bash
hdfs dfs -rm -r /user/root/wordcount/output 
```
  
5. Comprobar el resultado 
```bash
hdfs dfs -cat /user/root/wordcount/output/part-r-00000 
```

Aquí vemos como nos ha contado las palabras de nuestro archivo en el cual teníamos: 
"Hola mundo Hola world" 

![img2](https://tajamar365-my.sharepoint.com/:i:/p/tsuenkit_lui/EYEZOmeT-sdOj1uGzJtq78gBKqxB1hBZm9DVgnwFs5SwtA?e=xCCrGr)