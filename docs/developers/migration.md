# Migracion de version 1.0 de andino a 2.0

En el presente documento se pretende explicar como llevar a cabo una migracion de la version 1.0 de andino a la version 2.0 de andino.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Requisitos](#requisitos)
- [Script](#script)
- [1) Backups](#1-backups)
  - [1.1) Base de datos](#11-base-de-datos)
  - [1.2) Archivos de la aplicacion](#12-archivos-de-la-aplicacion)
- [2) Instalación](#2-instalaci%C3%B3n)
  - [2.1) Detener la aplicación](#21-detener-la-aplicaci%C3%B3n)
  - [2.2) Instalar la aplicación](#22-instalar-la-aplicaci%C3%B3n)
- [3) Restores](#3-restores)
  - [3.1) Restaurar los archivos](#31-restaurar-los-archivos)
  - [3.2) Restaurar la base de datos](#32-restaurar-la-base-de-datos)
  - [3.3) Regenerar el índice de búsquedas](#33-regenerar-el-%C3%ADndice-de-b%C3%BAsquedas)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Requisitos

Se requiere tener instalado:

- [jq](https://stedolan.github.io/jq/download/) >= 1.5
- docker
- docker-compose


Se asume que en el servidor hay 3 containers de docker corriendo:

- `app-ckan`
- `pg-ckan`
- `solr-ckan`

Ademas se debe conocer los `usuarios` y `passwords` de la base de datos (tanto de la usada por `ckan` como por el `datastore`).

## Script de migración automático.

El repositorio cuenta con un script para correr la migración automáticamente.
El mismo se puede encontrar en [`install/migrate.sh`](https://github.com/datosgobar/portal-andino/blob/master/install/migrate.sh), dentro del repositorio.
Ciertas variables de entorno y tener instalado `docker` y `docker-compose`.
Debe ser ejecutado con `sudo` o `root`.

Ejemplo:

    export EMAIL=admin@example.com
    export HOST=andino.midomionio.com.ar
    export DB_USER=usuario
    export DB_PASS=password
    export STORE_USER=dsuser
    export STORE_PASS=dspass
    sudo -E ./migrate.sh


## Migración manual

Para realizar la migracion manual, debemos conocer las variables con las que se inicializo el portal.
En este caso, seran las siguientes:

    export EMAIL=admin@example.com
    export HOST=andino.midomionio.com.ar
    export DB_USER=usuario
    export DB_PASS=password
    export STORE_USER=dsuser
    export STORE_PASS=dspass

### 1.1) Backup de la Base de datos

Es necesario hacer un backup de la base de datos antes de empezar con la migración. La misma puede llevarse a cabo con el siguiente script:

```bash
#!/usr/bin/env bash
set -e;

old_db="pg-ckan"
database_backup="backup.gz"

echo "Creando backup de la base de datos."

backupdir=$(mktemp -d)

backupfile="$backupdir/$database_backup"
echo "Iniciando backup de $old_db"
echo "Usando directorio temporal: $backupdir"
docker exec $old_db pg_dumpall -c -U postgres | gzip > "$backupfile"
echo "Copiando backup a $PWD"

cp "$backupfile" $PWD
echo "Backup listo."
```

Este script dejara un archivo `backup.gz` en el directorio actual.

### 1.2) Backup de los archivos de la aplicacion

Es necesario hacer un backup de los archivos de la aplicacion: configuracion y archivos subidos. El mismo puede llevarse a cabo con el siguiente script:

**Nota:** Requiere [jq](https://stedolan.github.io/jq/) >= 1.5

```bash
#!/usr/bin/env bash
set -e;

old_andino="app-ckan"
app_backup="backup.tar.gz"

echo "Creando backup de los archivos de configuración."
backupdir=$(mktemp -d)
today=`date +%Y-%m-%d.%H:%M:%S`
appbackupdir="$backupdir/application/"
mkdir $appbackupdir
echo "Iniciando backup de los volumenes en $old_andino"
echo "Usando directorio temporal: $backupdir"
docker inspect --format '{{json .Mounts}}' $old_andino  | jq -r '.[]|[.Name, .Source, .Destination] | @tsv' |
while IFS=$'\t' read -r name source destination; do
    echo "Guardando archivos de $destination"
    if ls $source/* 1> /dev/null 2>&1; then
        echo "Nombre del volumen: $name."
        echo "Directorio en el Host: $source"
        echo "Destino: $destination"
        dest="$appbackupdir$name"
        mkdir -p $dest
        echo "$destination" > "$dest/destination.txt"

        tar -C "$source" -zcvf "$dest/backup_$today.tar.gz" $(ls $source)
        echo "List backup de $destination"
    else
        echo "Ningún archivo para $destination";
    fi
done
echo "Generando backup en $app_backup"
tar -C "$appbackupdir../" -zcvf $app_backup "application/"
echo "Backup listo."
```

Este script dejara un archivo backup.tar.gz en el directorio actual. El mismo, una vez descomprimido, contendra la siguiente estructura (por ejemplo):


    - application/
        ├── 61ee6cc7dc974476fe3300cc4325d913ed2f949494419b11a5c7c897fa919106
        │   ├── backup_2017-05-19.10:56:09.tar.gz
        │   └── destination.txt
        └── b1bf820976c3220e54136e4db229a67a9d9292896ad8d91623030e3b7171f210
            ├── backup_2017-05-19.10:56:09.tar.gz
            └── destination.txt

Cada sub-directorio contiene el ID del volumen en docker usado, los numero varian de volumen en volumen. Dentro de cada sub-directorio se encuentra un archivo *.tar.gz junto con un archivo destination.txt. El archivo destination.txt indica donde corresponde la informacion dentro del container, el archivo *.tar.gz contiene una carpeta _data con los archivos.

### 2.1) Detener la aplicación

Debemos detener la aplicacion para lograr que se liberen los puertos usados, por ejemplo el puerto 80.

docker stop solr-ckan pg-ckan app-ckan

### 2.2) Instalar la aplicación

Ver la documentación [Aquí](install.md)

**Nota:** Actualizar la version de docker y docker-compose de ser necesario.

Ahora es necesario restaurar tanto la base de datos como los archivos de la aplicacion.

### 3.1) Restaurar los archivos

Descomprimir el archivo `backup.tar.gz`. En cada subdirectorio encontraremos el archivo destination.txt, el contenido de este archivo nos ayudara a saber donde debemos copiar los archivos. Con el siguiete comando podremos saber que volumenes hay montados en el nuevo esquema y donde debemos copiar los archivos dentro del `backup_*.tar.gz`

Correr `docker inspect andino -f '{{ json .Mounts }}' | jq`:

El comando mostrará lo siquiente, por ejemplo:

    [
    {
        "Type": "volume",
        "Name": "a1d87160a04e270302582849c9ce5c6dbb44719a94b702158aeaf23835f7862f",
        "Source": "/var/lib/docker/volumes/a1d87160a04e270302582849c9ce5c6dbb44719a94b702158aeaf23835f7862f/_data",
        "Destination": "/etc/ckan/default",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    },
    {
        "Type": "volume",
        "Name": "7ab721966628bf692a3d451567c9a01b419ba5189b88ef05484de315c73f6275",
        "Source": "/var/lib/docker/volumes/7ab721966628bf692a3d451567c9a01b419ba5189b88ef05484de315c73f6275/_data",
        "Destination": "/usr/lib/ckan/default/src/ckanext-gobar-theme/ckanext/gobar_theme/public/user_images",
        "Driver": "local",
        "Mode": "",
        "RW": true,
        "Propagation": ""
    },

    ...

Como podemos ver, hay una entrada "Destination" que coincidira con el contenido del archivo destination.txt en cada directorio. Debemos asegurarnos de no copiar el archivo production.ini, ya que el mismo cambio bastante de version en version.

El restore puede ser llevado a cabo con el siguiente script:

```bash
#!/usr/bin/env bash
set -e;

echo "Iniciando recuperación de Archivos."
install_dir="/etc/portal";
container="andino"
app_backup="backup.tar.gz"

containers=$(docker ps -q)
if [ -z "$containers" ]; then
    echo "No se encontró ningun contenedor corriendo."
else
    docker stop $containers
fi

restoredir=$(mktemp -d)
echo "Usando directorio temporal $restoredir"
tar zxvf $app_backup -C $restoredir

docker inspect --format '{{json .Mounts}}' $container  | jq -r '.[]|[.Name, .Source, .Destination] | @tsv' |
while IFS=$'\t' read -r name source destination; do
    for directory in $restoredir/application/*; do
        dest=$(cat "$directory/destination.txt")
        if [ "$dest" == "$destination" ]; then
            echo "Recuperando archivos para $destination"
            tar zxvf "$directory/$(ls "$directory" | grep backup)" -C "$source"
        fi
    done
done
echo "Restauración lista."
echo "Reiniciando servicios."
cd $install_dir;
docker-compose -f latest.yml restart;
cd -;
```


### 3.2) Restaurar la base de datos

Para restaurar la base de datos se puede usar el siguiente script contra el archivo previamente generado (backup.gz):

```bash
#!/usr/bin/env bash
set -e;

install_dir="/etc/portal";
database_backup="backup.gz"
container="andino-db"

echo "Iniciando restauración de la base de datos."
containers=$(docker ps -q)

if [ -z "$containers" ]; then
    echo "No se encontró ningun contenedor corriendo."
else
    docker stop $containers
fi
docker restart $container
sleep 10;

restoredir=$(mktemp -d);
echo "Usando directorio temporal $restoredir"

restorefile="$restoredir/dump.sql";

gzip -dkc < $database_backup > "$restorefile";
echo "Borrando base de datos actual."
docker exec $container psql -U postgres -c "DROP DATABASE IF EXISTS ckan;"
docker exec $container psql -U postgres -c "DROP DATABASE IF EXISTS datastore_default;"
echo "Restaurando la base de datos desde: $restorefile"
cat "$restorefile" | docker exec -i $container psql -U postgres
echo "Recuperando credenciales de los usuarios"
docker exec  $container psql -U postgres -c "ALTER USER $DB_USER WITH PASSWORD '$DB_PASS';"
docker exec  $container psql -U postgres -c "ALTER USER $STORE_USER WITH PASSWORD '$STORE_PASS';"

echo "Restauración lista."
echo "Reiniciando servicios."
cd $install_dir;
docker-compose -f latest.yml restart;
cd -;
```


### 3.3) Regenerar el índice de búsquedas

Para regenerar el índice de búsquedas, debemos ir al directorio donde se instaló la aplicación y correr el siguiente comando:

    docker-compose -f latest.yml exec portal /etc/ckan_init.d/run_rebuild_search.sh
