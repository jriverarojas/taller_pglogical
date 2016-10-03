# Taller Pglogical
# 1. Preparando Entorno
Ejecutar :
```
yum -y install wget
wget http://yum.postgresql.org/9.4/redhat/rhel-7-x86_64/pgdg-centos94-9.4-2.noarch.rpm
rpm -Uvh --replacepkgs pgdg-centos94-9.4-2.noarch.rpm
yum -y install postgresql94-server postgresql94-docs postgresql94-contrib postgresql94-plperl postgresql94-plpython postgresql94-pltcl postgresql94-test rhdb-utils gcc-objc postgresql94-devel

/usr/pgsql-9.4/bin/postgresql94-setup initdb

wget http://yum.postgresql.org/9.5/redhat/rhel-7-x86_64/pgdg-centos95-9.5-2.noarch.rpm
rpm -Uvh --replacepkgs pgdg-centos95-9.5-2.noarch.rpm
yum -y install postgresql95-server postgresql95-docs postgresql95-contrib postgresql95-plperl postgresql95-plpython postgresql95-pltcl postgresql95-test rhdb-utils gcc-objc postgresql95-devel

/usr/pgsql-9.5/bin/postgresql95-setup initdb
```
Cambiar en archivo de configuracion /var/lib/pgsql/9.4/data/postgresql.conf :
```
listen_address    *
```
Cambiar en archivo de configuracion /var/lib/pgsql/9.5/data/postgresql.conf :
```
listen_address    *
port              5433
```
Adicionar el rango de la red local en el archivo /var/lib/pgsql/9.4/data/pg_hba.conf . Ejemplo:
```
host    all		all     	192.168.56.0/24         md5
```
Adicionar el rango de la red local en el archivo /var/lib/pgsql/9.5/data/pg_hba.conf . Ejemplo:
```
host    all		all     	192.168.56.0/24         md5
```
Ejecutar :
```
systemctl start postgresql-9.4
systemctl enable postgresql-9.4
systemctl start postgresql-9.5
systemctl enable postgresql-9.5
```
Cambiar passwords para usuario Postgres en ambos clusters, creacion de base de datos y tabla

```
su postgres
psql
alter user postgres password 'postgres';
CREATE DATABASE dbprueba
  WITH OWNER = postgres
    ENCODING = 'UTF8';
\connect dbprueba
CREATE TABLE tprueba
(
  id integer,
  texto text,
  CONSTRAINT prueba_pkey PRIMARY KEY(id)
);
\q

psql -p 5433
alter user postgres password 'postgres';
CREATE DATABASE dbprueba
  WITH OWNER = postgres
    ENCODING = 'UTF8';
\connect dbprueba
CREATE TABLE tprueba
(
  id integer,
  texto text,
  CONSTRAINT prueba_pkey PRIMARY KEY(id)
);
\q
exit
```
# 2. Instalando Pglogical
Instalar pglogical. Ejecutar:

```
yum -y install http://packages.2ndquadrant.com/pglogical/yum-repo-rpms/pglogical-rhel-1.0-2.noarch.rpm

yum -y install postgresql95-pglogical
yum -y install postgresql94-pglogical
```
# 3. Configurando postgresql.conf y pg_hba.conf
Cambiar en archivo de configuracion /var/lib/pgsql/9.4/data/postgresql.conf :
```
wal_level                 logical 
max_worker_processes      10 
max_replication_slots     10 
max_wal_senders           10 
shared_preload_libraries  'pglogical'
```

Cambiar en archivo de configuracion /var/lib/pgsql/9.5/data/postgresql.conf :
```
wal_level                 logical 
max_worker_processes      10 
max_replication_slots     10 
shared_preload_libraries  'pglogical'
```
Adicionar el rango de la red local en el archivo /var/lib/pgsql/9.4/data/pg_hba.conf . Ejemplo:
```
host    replication		replication     	192.168.56.0/24         md5
```
Adicionar el rango de la red local en el archivo /var/lib/pgsql/9.5/data/pg_hba.conf . Ejemplo:
```
host    replication		replication     	192.168.56.0/24         md5
```
Aplicando cambios en archivos de configuracion. Ejecutar:
```
systemctl restart postgresql-9.4
systemctl restart postgresql-9.5
```

# 4. Creando extensiones
En pgadmin ejecutar como consulta para el puerto 5432 (MASTER):
```
CREATE EXTENSION pglogical_origin;
CREATE EXTENSION pglogical;
CREATE ROLE replication WITH SUPERUSER REPLICATION LOGIN ENCRYPTED PASSWORD 'replication';
```

Para el puerto 5433 (SLAVE):
```
CREATE EXTENSION pglogical;
CREATE ROLE replication WITH SUPERUSER REPLICATION LOGIN ENCRYPTED PASSWORD 'replication';
```
# 5. Configurando replicacion

Ejecutar en pgadmin como consulta en la bd master puerto 5432:
```
SELECT pglogical.create_node(
    node_name := 'master',
    dsn := 'host=192.168.56.10 user=replication password=replication port=5432 dbname=dbprueba'
);

select pglogical.create_replication_set('replicacion');

select pglogical.replication_set_add_table('replicacion','public.tprueba', true);
```

Ejecutar en pgadmin como consulta en la bd master puerto 5433:
```
SELECT pglogical.create_node(
    node_name := 'slave',
    dsn := 'host=192.168.56.10 user=replication password=replication port=5433 dbname=dbprueba'
);

SELECT pglogical.create_subscription(
    subscription_name := 'subscripcion',
    provider_dsn := 'user=replication host=192.168.56.10 port=5432 password=replication dbname=dbprueba' 
);

select pglogical.alter_subscription_add_replication_set('subscripcion', 'replicacion');
```

Para poder ver el estado de la replicacion en slave se puede ejecutar :
```
select * FROM pglogical.show_subscription_status();
```

PROBAR REPLICACION

# 6. Eliminando replicacion
Para eliminar en slave ejecutar:
```
select pglogical.alter_subscription_remove_replication_set('subscripcion', 'replicacion');
select pglogical.drop_subscription('subscripcion');
select pglogical.drop_node('slave');
```
Para eliminar en master ejecutar:
```
select pglogical.replication_set_remove_table('replicacion','public.tprueba')
select pglogical.drop_replication_set('replicacion'); 
select pglogical.drop_node('master');
```
