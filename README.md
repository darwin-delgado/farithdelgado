# farithdelgado
db-backup-nfs:/originadores/DATABASE /DATABASE nfs rw,hard,noatime,nodiratime,_netdev,nofail 0 0


###### Scrip
docker exec -i <contenedor> mysql -u root -p -e "ALTER USER 'us.backup'@'localhost' IDENTIFIED BY 'Privacy2024*+'; FLUSH PRIVILEGES;"

docker run --rm -it -v <volumen_mysql>:/var/lib/mysql mysql:8.0 --skip-grant-table

# Detener el contenedor                                                                                                                                                                     
  docker stop <contenedor>                                                                                                                                                                    
                                                                                                                                                                                              
  # Arrancar temporal sin autenticación
  docker run --rm -d --name mysql-reset \                                                                                                                                                     
    -v landing_mysql_data:/var/lib/mysql \
    mysql:8.0 --skip-grant-tables --skip-networking

  # Entrar y resetear password
  docker exec -it mysql-reset mysql -u root -e "FLUSH PRIVILEGES; ALTER USER 'root'@'localhost' IDENTIFIED BY 'Privacy2024*+'; FLUSH PRIVILEGES;"

  # Detener el temporal
  docker stop mysql-reset

  # Arrancar el contenedor original
  docker start <contenedor>


docker exec -it mysql-reset mysql -u root -e "
  FLUSH PRIVILEGES;
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Privacy2024*+';
  FLUSH PRIVILEGES;"

docker exec -it mysql-reset mysql -u root -e "SELECT user, host, plugin, authentication_string FROM mysql.user WHERE user='root';"


+------+-----------+-----------------------+-------------------------------------------+
| user | host      | plugin                | authentication_string                     |
+------+-----------+-----------------------+-------------------------------------------+
| root | localhost | mysql_native_password | *D8C1F2F50C2EF1543896C4F01C6A33C230C8E102 |
+------+-----------+-----------------------+-------------------------------------------+


docker exec -it mysql-reset mysql -u root -e "                                                                                                                                              
  UPDATE mysql.user SET authentication_string='' WHERE user='root' AND host='localhost';
  FLUSH PRIVILEGES;                                                                                                                                                                           
  ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'Privacy2024*+';
  FLUSH PRIVILEGES;"

