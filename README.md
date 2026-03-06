# farithdelgado
db-backup-nfs:/originadores/DATABASE /DATABASE nfs rw,hard,noatime,nodiratime,_netdev,nofail 0 0


###### Scrip

#!/bin/bash
set -e

# Variables
DIR=$(dirname "$0")/mysql
MYSQLDUMP=mysqldump
NAME_DOCKER=db-landing
DATABASE=comunidad-all
DB_USER=us.backup
BACKUP_EXT="sql"
BACKUP_FILE="$DIR/$NAME_DOCKER-$(date +\%F).$BACKUP_EXT"
COMPRESSED_FILE="$BACKUP_FILE.gz"
BACKUP_DIR="/DATABASE/$(hostname)/mysql"
BACKUP_KEEP=2 #Cantidad de últimos backups a mantener.

ENABLE_MTTO=0
MYSQLCHECK=mysqlcheck
DAY_MTTO=6 #Realizar el mantenimiento los sábados


# Función para obtener la fecha y hora actual
date_now() {
    date "+%Y-%m-%d %H:%M:%S"
}

echo "Proceso iniciado $(date_now)"


# Función para realizar el backup
run_backup() {
    echo "  Inicio de backup. $(date_now)"

        docker cp ~/.my.cnf $NAME_DOCKER:/etc/mysql/conf.d
        docker exec -i $NAME_DOCKER bash -c "cd /etc/mysql/conf.d; chown root:root .my.cnf; chmod 600 .my.cnf;"

    if ! docker exec -i $NAME_DOCKER bash -c "$MYSQLDUMP --single-transaction --quick $DATABASE" > "$BACKUP_FILE"; then
        printf "Error: Falló la generación del backup con $MYSQLDUMP. $(date_now)\n" >&2
        return 1
    fi
    if ! ls -lh "$BACKUP_FILE" | awk '{print "  Archivo generado: "$5}'; then
        printf "Error: No se pudo verificar el archivo generado. $(date_now)\n" >&2
        return 1
    fi
}

# Función para comprimir el backup
compress_backup() {
    echo "  Inicio de compresión. $(date_now)"
    if ! gzip -f "$BACKUP_FILE"; then
        printf "Error: Falló la compresión del archivo. $(date_now)\n" >&2
        return 2
    fi
    if ! ls -lh "$COMPRESSED_FILE" | awk '{print "  Archivo comprimido: "$5}'; then
        printf "Error: No se pudo verificar el archivo comprimido. $(date_now)\n" >&2
        return 2
    fi
}


# Función para copiar a la red
copy_to_network() {
    echo "  Copiando a la red. $(date_now)"
    mkdir -p "$BACKUP_DIR"
    if ! rsync -avzq "$COMPRESSED_FILE" "$BACKUP_DIR"; then
        printf "Error: Falló la copia a la red. $(date_now)\n" >&2
        return 3
    fi
}

# Función para limpiar respaldos antiguos
clean_old_backups() {
        #Valor por defecto: 2
        BACKUP_KEEP=${BACKUP_KEEP:-2}

    echo "  Limpiando respaldos antiguos. $(date_now)"

    # Definir los patrones a limpiar
    local patterns=("$NAME_DOCKER-*.$BACKUP_EXT.gz" "$NAME_DOCKER-*.log")

    # Limpiar ambos directorios para cada patrón
    for dir in "$DIR" "$BACKUP_DIR"; do
        for pattern in "${patterns[@]}"; do
            if ! clean_directory "$dir" "$pattern" "$BACKUP_KEEP"; then
                return 4
            fi
        done
    done
}

# Función para limpiar archivos antiguos en un directorio específico
clean_directory() {
    local dir=$1
    local pattern=$2
    local keep=$3

    # Verificar que podemos acceder al directorio
        if [ ! -d "$dir" ]; then
        printf "Error: Directorio $dir no existe." >&2
        return 1
    fi

    local total_files=$(ls -t "$dir"/$pattern 2>/dev/null | wc -l)

    if [ "$total_files" -gt "$keep" ]; then
        if ! ls -t "$dir"/$pattern 2>/dev/null | tail -n +$((keep + 1)) | xargs -r rm -f; then
            printf "Error: Falló al limpiar archivos $pattern en $dir. $(date_now)\n" >&2
            return 2
        fi
    fi
    return 0
}

# Función para realizar mantenimiento a las bases de datos
run_mtto() {
        #Solo realizar el mantenimiento si el dia de la semana es igual a $DAY_MTTO
        if [ $(date '+%w') -eq $DAY_MTTO ]; then
                printf "  Iniciando mantenimiento $(date_now)\n" >&2
                #$MYSQLCHECK --optimize --databases $DATABASE
                if ! docker exec -i $NAME_DOCKER $MYSQLCHECK --optimize --all-databases > "$DIR/$DATABASE-mtto-$(date +\%F).log"; then
                        printf "Error: Falló mantenimiento con $MYSQLCHECK. $(date_now)\n" >&2
                        return 5
                fi
        fi
}

# Ejecución de las funciones con manejo de errores
main() {
    if ! run_backup; then
        printf "Error: La función run_backup falló.\n" >&2
        exit 1
    fi

    if ! compress_backup; then
        printf "Error: La función compress_backup falló.\n" >&2
        exit 2
    fi

    if ! copy_to_network; then
        printf "Error: La función copy_to_network falló.\n" >&2
        exit 3
    fi

    if ! clean_old_backups; then
        printf "Error: La función clean_old_backups falló.\n" >&2
        exit 4
    fi

        if [ "$ENABLE_MTTO" -ne 0 ]; then
                if ! run_mtto; then
                        printf "Error: La función run_mtto falló.\n" >&2
                        exit 5
                fi
        else
        printf "  Mantenimiento deshabilitado.\n" >&2
    fi
}

main
echo "Proceso terminado $(date_now)"

[us.backup@BUDCK1 backups]$  cat backup-mysql-db_app.sh
#!/bin/bash
set -e

# Variables
DIR=$(dirname "$0")/mysql
MYSQLDUMP=mysqldump
NAME_DOCKER=db_app
DATABASE=kb_app
DB_USER=us.backup
BACKUP_EXT="sql"
BACKUP_FILE="$DIR/$NAME_DOCKER-$(date +\%F).$BACKUP_EXT"
COMPRESSED_FILE="$BACKUP_FILE.gz"
BACKUP_DIR="/DATABASE/$(hostname)/mysql"
BACKUP_KEEP=2 #Cantidad de últimos backups a mantener.

ENABLE_MTTO=0
MYSQLCHECK=mysqlcheck
DAY_MTTO=6 #Realizar el mantenimiento los sábados


# Función para obtener la fecha y hora actual
date_now() {
    date "+%Y-%m-%d %H:%M:%S"
}

echo "Proceso iniciado $(date_now)"


# Función para realizar el backup
run_backup() {
    echo "  Inicio de backup. $(date_now)"

        docker cp ~/.my.cnf $NAME_DOCKER:/etc/mysql/conf.d
        docker exec -i $NAME_DOCKER bash -c "cd /etc/mysql/conf.d; chown root:root .my.cnf; chmod 600 .my.cnf;"

    if ! docker exec -i $NAME_DOCKER bash -c "$MYSQLDUMP --single-transaction --quick $DATABASE" > "$BACKUP_FILE"; then
        printf "Error: Falló la generación del backup con $MYSQLDUMP. $(date_now)\n" >&2
        return 1
    fi
    if ! ls -lh "$BACKUP_FILE" | awk '{print "  Archivo generado: "$5}'; then
        printf "Error: No se pudo verificar el archivo generado. $(date_now)\n" >&2
        return 1
    fi
}

# Función para comprimir el backup
compress_backup() {
    echo "  Inicio de compresión. $(date_now)"
    if ! gzip -f "$BACKUP_FILE"; then
        printf "Error: Falló la compresión del archivo. $(date_now)\n" >&2
        return 2
    fi
    if ! ls -lh "$COMPRESSED_FILE" | awk '{print "  Archivo comprimido: "$5}'; then
        printf "Error: No se pudo verificar el archivo comprimido. $(date_now)\n" >&2
        return 2
    fi
}


# Función para copiar a la red
copy_to_network() {
    echo "  Copiando a la red. $(date_now)"
    mkdir -p "$BACKUP_DIR"
    if ! rsync -avzq "$COMPRESSED_FILE" "$BACKUP_DIR"; then
        printf "Error: Falló la copia a la red. $(date_now)\n" >&2
        return 3
    fi
}

# Función para limpiar respaldos antiguos
clean_old_backups() {
        #Valor por defecto: 2
        BACKUP_KEEP=${BACKUP_KEEP:-2}

    echo "  Limpiando respaldos antiguos. $(date_now)"

    # Definir los patrones a limpiar
    local patterns=("$NAME_DOCKER-*.$BACKUP_EXT.gz" "$NAME_DOCKER-*.log")

    # Limpiar ambos directorios para cada patrón
    for dir in "$DIR" "$BACKUP_DIR"; do
        for pattern in "${patterns[@]}"; do
            if ! clean_directory "$dir" "$pattern" "$BACKUP_KEEP"; then
                return 4
            fi
        done
    done
}

# Función para limpiar archivos antiguos en un directorio específico
clean_directory() {
    local dir=$1
    local pattern=$2
    local keep=$3

    # Verificar que podemos acceder al directorio
        if [ ! -d "$dir" ]; then
        printf "Error: Directorio $dir no existe." >&2
        return 1
    fi

    local total_files=$(ls -t "$dir"/$pattern 2>/dev/null | wc -l)

    if [ "$total_files" -gt "$keep" ]; then
        if ! ls -t "$dir"/$pattern 2>/dev/null | tail -n +$((keep + 1)) | xargs -r rm -f; then
            printf "Error: Falló al limpiar archivos $pattern en $dir. $(date_now)\n" >&2
            return 2
        fi
    fi
    return 0
}

# Función para realizar mantenimiento a las bases de datos
run_mtto() {
        #Solo realizar el mantenimiento si el dia de la semana es igual a $DAY_MTTO
        if [ $(date '+%w') -eq $DAY_MTTO ]; then
                printf "  Iniciando mantenimiento $(date_now)\n" >&2
                #$MYSQLCHECK --optimize --databases $DATABASE
                if ! docker exec -i $NAME_DOCKER $MYSQLCHECK --optimize --all-databases > "$DIR/$DATABASE-mtto-$(date +\%F).log"; then
                        printf "Error: Falló mantenimiento con $MYSQLCHECK. $(date_now)\n" >&2
                        return 5
                fi
        fi
}

# Ejecución de las funciones con manejo de errores
main() {
    if ! run_backup; then
        printf "Error: La función run_backup falló.\n" >&2
        exit 1
    fi

    if ! compress_backup; then
        printf "Error: La función compress_backup falló.\n" >&2
        exit 2
    fi

    if ! copy_to_network; then
        printf "Error: La función copy_to_network falló.\n" >&2
        exit 3
    fi

    if ! clean_old_backups; then
        printf "Error: La función clean_old_backups falló.\n" >&2
        exit 4
    fi

        if [ "$ENABLE_MTTO" -ne 0 ]; then
                if ! run_mtto; then
                        printf "Error: La función run_mtto falló.\n" >&2
                        exit 5
                fi
        else
        printf "  Mantenimiento deshabilitado.\n" >&2
    fi
}

main
echo "Proceso terminado $(date_now)"
