Alterar senha de usuário, 2 caminhos:
 -SET PASSWORD FOR 'user'@'%' = PASSWORD('nova_senha');
 -mysqladmin -u user -p nova_senha senha_antiga

Recuperar senha de root
 -mysqld_safe --skip-grant-tables & 
 -mysql> use mysql; 
 -mysql> update user SET password=password('NOVA SENHA') WHERE user='root';

Apagar todas tabelas de um banco 
 -mysqldump -uuser --no-data dbname -p'senha'  
   | grep ^DROP | mysql -uuser dbname -p'senha'

Criar banco por Shell
 -mysqladmin -u root -p create dbname

Apagar banco Shell
 -mysqladmin -u root -p drop dbname

Dump MySQL
 -Monta lista com nomes dos dbs
  mysql -uthiago -psenha 
    -e "SHOW DATABASES" | 
     grep -v 'Database' | 
     grep -v 'information_schema' | 
     grep -v 'phpmyadmin' | 
     grep -v 'mysql' 
     >> /tmp/lista.txt
  -Roda o Dump
    now=$(date -d 'today' '+%Y.%m.%d.%H.%m.%s')
    for line in $(cat lista.txt); 
    do  
      mysqldump -uthiago $line -psenha | gzip > /dumps/$line.$now.sql.gz
    done;




mifra mysql server users and dbs:

#!/bin/bash

# Copiado de: Author Vivek Gite <vivek@nixcraft.com>
# esse arquivo deve estar no servidor de origem
# copia banco e usuário+permissões local para um servidor remoto
# rodar assim: migrate.sh nomedobanco nome_usuario (o segundo parametro é opicional, normalmente user é igual ao dbname)

echo "Iniciado em:"
date

cd $(dirname $0)
source settings.sh

migrate () {
  # Input data
  _db="$1"
  _user="$2"

  # pasta com permissão de escrita
  _tmp="/tmp/output.mysql.$$.sql"
  _dump="/tmp/output.mysql.dump.$_db.$$.sql"

  # Die if no input given
  [ $# -eq 0 ] && { echo "Usage: $0 MySQLDatabaseName MySQLUserName"; exit 1; }

  #if no user is given, we use the db value instead
  if [ $# -eq 1 ] 
  then
   _user=${_db}
  fi

  # Make sure you can connect to local db server
  mysqladmin -u "$_lusr" -p"$_lpass" -h "$_lhost"  ping &>/dev/null || { echo "Error: Mysql server is not online or set correct values for _lusr, _lpass, and _lhost"; exit 2; }
 
  # Make sure database exists
  mysql -u "$_lusr" -p"$_lpass" -h "$_lhost" -N -B  -e'show databases;' | grep -q "^${_db}$" ||  { echo "Error: Database $_db not found."; exit 3; }
 
  ##### Step 1: Okay build .sql file with db and users, password info ####
  echo "*** Getting info about $_db..."
  echo "create database IF NOT EXISTS $_db; " > "$_tmp"
 
  # Build mysql query to grab all privs and user@host combo for given db_username
  mysql -u "$_lusr" -p"$_lpass" -h "$_lhost" -B -N \
  -e "SELECT DISTINCT CONCAT('SHOW GRANTS FOR ''',user,'''@''',host,''';') AS query FROM user WHERE user != '' " \
  mysql \
  | mysql  -u "$_lusr" -p"$_lpass" -h "$_lhost" \
  | grep  "$_user" \
  | sed ':a;N;$!ba;s/\n/;\n/g' \
  | sed ':a;N;$!ba;s/\\\\//g' \
  | sed ':a;N;$!ba;s/localhost/'${_lhostname}'/g' \
  | sed 's/Grants for .*/#### &/' >> "$_tmp"

  #### Step 2: Create db and load users into remote db server ####
  mysql -u "$_rusr" -p"$_rpass" -h"$_rhost" < "$_tmp"
 
  #### Step 3: Send mysql database and all data ####
  echo "*** Exporting $_db from ${_lhost} and importing in ${_rhost}..."
  mysqldump -u "$_lusr" "$_db" -p"$_lpass" -h"$_lhost" > "$_dump"
  mysql -u "$_rusr" "$_db" -p"$_rpass" -h"$_rhost" < "$_dump"
 
  rm -f "$_tmp"
}

# Buscar pelo nomes dos bancos, vamos supor que os nomes dos usuário são os mesmos
dbs=$(mysql -u "$_lusr" -p"$_lpass" -h "$_lhost" -e "SHOW DATABASES" \
| grep -v 'Database' | grep -v 'information_schema' \
| grep -v 'mysql' | grep -v 'phpmyadmin')

for db in $dbs;
do
  user=$db
  echo "--------------- Migrando $db -----------------"
  migrate $db $user
done

echo "Finalizado em:"
date


drop mysql tables:

#!/bin/bash
# Part of this code was took from: http://www.cyberciti.biz/faq/how-do-i-empty-mysql-database/
MUSER="$1"
MPASS="$2"
MDB="$3"
MHOST="$4"
 
# Detect paths
MYSQL=$(which mysql)
AWK=$(which awk)
GREP=$(which grep)
 
if [ $# -ne 4 ]
then
	echo "Usage: $0 {MySQL-User-Name} {MySQL-User-Password} {MySQL-Database-Name} {host}"
	echo "Drops all tables from a MySQL"
	exit 1
fi
 
TABLES=$($MYSQL -u $MUSER -h$MHOST -p$MPASS $MDB -e 'show tables' | $AWK '{ print $1}' | $GREP -v '^Tables' )
 
for t in $TABLES
do
	echo "Deleting $t table from $MDB database..."
	$MYSQL -u $MUSER -h$MHOST -p$MPASS $MDB -e "drop table $t"
done


##################
#!/bin/bash

# mkdir -p /bkp/mysql
# exemplo de crontab: 29 0 * * * /home/thiago/repos/archive/scripts/mysql/backup_all_databases.sh root senha localhost

DIR_BKP='/home/thiago/data/bkp/mysql'
_usuario="$1"
_senha="$2"
_maquina="$3"

if [ $# -ne 3 ]
then
	echo "Usage: $0 {MySQL-ADMIN-User-Name} {MySQL-User-Password}  {host}"
	exit 1
fi

if [ ! -d "$DIR_BKP" ]
then
    mkdir -p $DIR_BKP
fi

# data atual
DATE=`date +%Y%m%d%H%M%S`

# Monta o arquivo lista:
lista=$(mysql -u"$_usuario" -h "$_maquina" -p"$_senha" -e "SHOW DATABASES" | grep -v 'Database' | grep -v 'information_schema' | grep -v 'mysql')

#faz o backup de cada banco
for db in $lista
do
  mysqldump -u"$_usuario" $db -p"$_senha" -h "$_maquina" --opt --single-transaction  | gzip > $DIR_BKP/$db$DATE.sql.gz
done

mysqldump -u"$_usuario" -p"$_senha" -h"$_maquina" --events --skip-lock-tables --all-databases --opt --single-transaction  > $DIR_BKP/alldatabases$DATE.sql

# TODO: Apagar backups antigos na mesma rotina