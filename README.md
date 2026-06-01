# Replicació de sistemes heterogenis

## Introducció

Treballarem amb dos sistemes de bases de dades diferents, un és MySQL i l'altre és PostgreSQL, he preparat una base de dades `employees` a MySQL i després l’he migrat a PostgreSQL fent servir `pgloader` i després he configurat SymmetricDS per provar la replicació entre les dos bases de dades. La pràctica l’he dividit en tres parts primer la migració inicial de MySQL a PostgreSQL amb pgloader, després replicació principal-secundari: MySQL a PostgreSQL i l'últim replicació principal-principal: MySQL a PostgreSQL i PostgreSQL a MySQL.

## Entorn utilitzat

He fet servir dos màquines Ubuntu Server 24.04 dintre de VirtualBox, la primera màquina és la de MySQL:

```text
Nom: asgbd-mysql-oc
IP interna: 192.168.1.112
Sistema: Ubuntu Server 24.04
Servei principal: MySQL
```

La segona màquina és la de PostgreSQL:

```text
Nom: asgbd-psql-oc
IP interna: 192.168.1.113
Sistema: Ubuntu Server 24.04
Servei principal: PostgreSQL 18
```

Primer he comprovat que les màquines tenien la IP correcta i que els serveis principals estaven preparats.

![Entorn MySQL](capturas/01a_entorn_mysql.png)

Aqui tenim la màquina `asgbd-mysql-oc`, la seva IP `192.168.1.112` i la versió de MySQL instal·lada.

![Entorn PostgreSQL](capturas/01b_entorn_postgresql.png)

I aqui la màquina `asgbd-psql-oc` amb la IP `192.168.1.113`.

## Càrrega de la base de dades employees a MySQL

A la màquina MySQL he carregat la base de dades `employees`, que és la base de dades que després migraré a PostgreSQL i vaig carregar la base de dades, i aqui comprobo si s'ha fet bé:

```sql
SHOW DATABASES;
USE employees;
SHOW TABLES;
SELECT COUNT(*) FROM employees;
SELECT emp_no, first_name, last_name FROM employees LIMIT 5;
```

![Employees carregada a MySQL](capturas/02_mysql_employees_carregada.png)

La base `employees` existeix a MySQL, que té les taules carregades i que la taula `employees` té informació.

## Preparació de PostgreSQL 18

A la màquina PostgreSQL he instal·lat PostgreSQL 18, ara crearé una base de dades buida amb el nom `employees`, que serà la destinació de la migració:

```bash
sudo -u postgres dropdb --if-exists employees
sudo -u postgres createdb employees
sudo -u postgres psql -lqt | grep employees
```

Per permetre connexions remotes vaig modificar aquests fitxers:

```text
/etc/postgresql/18/main/postgresql.conf
/etc/postgresql/18/main/pg_hba.conf
```

Al `postgresql.conf` modifiquem la linea:

```conf
listen_addresses = '*'
```

I al `pg_hba.conf` afegim la xarxa interna a la última línea del ficher:

```conf
host    all     all     192.168.1.0/24     scram-sha-256
```

![PostgreSQL 18 preparat](capturas/03_postgresql_18_preparat.png)

PostgreSQL 18 està funcionant, la base `employees` existeix i que la màquina està preparada per rebre connexions de moment tot perfecte.

## Compilació de pgloader

L’enunciat diu que el `pgloader` dels repositoris podia donar problemes amb MySQL 8, per això he fet servir la versió de GitHub, preparant-la jo mateix a la màquina MySQL, primer he instal·lat les dependències que feian falta pero poder-ho fer:

```bash
sudo apt update
sudo apt install -y git curl make gcc sbcl unzip libsqlite3-dev gawk freetds-dev libzip-dev
```

Després vaig clonar-ho i compilar-ho:

```bash
git clone https://github.com/dimitri/pgloader.git
cd pgloader
make pgloader
```

![pgloader compilat](capturas/04_pgloader_compilat_correctament.png)

Aqui estic fent servir el `pgloader` compilat dintre de la carpeta `~/pgloader`, no la versió instal·lada amb `apt`.

## Migració de MySQL a PostgreSQL

Per fer la migració, primer vaig crear un usuari a MySQL perquè `pgloader` pogués llegir la base `employees`:

```sql
CREATE USER IF NOT EXISTS 'pgloader'@'localhost' IDENTIFIED BY 'pgloader';
GRANT ALL PRIVILEGES ON employees.* TO 'pgloader'@'localhost';
FLUSH PRIVILEGES;
```

Després vaig executar la migració des de la màquina MySQL:

```bash
./build/bin/pgloader mysql://pgloader:pgloader@localhost/employees postgresql://postgres:postgres@192.168.1.113/employees
```

Aquesta ordre llegeix la base de dades `employees` de MySQL i la copia a la base de dades `employees` de PostgreSQL.

![Migració amb pgloader](capturas/05_migracio_pgloader_correcta.png)

Podem veure com s'ha importat tot bé i que no hi ha errors.

## Verificació de la migració a PostgreSQL

Després de la migració entrem a PostgreSQL i comprovem que l’esquema `employees` es crea sense cap problema, dintre de la base `employees`.

```bash
sudo -u postgres psql -d employees
```

A PostgreSQL vaig comprovar, que s’ha creat l’esquema `employees`, que dintre hi ha les taules migrades i que la taula principal `employees.employees` tenia informació, basicament vui comprobar que no només s'ha creat l’estructura, sinó que també s'ha passat tot el contingut:

```sql
\dn
\dt employees.*
SELECT COUNT(*) FROM employees.employees;
```

![Esquema employees a PostgreSQL](capturas/06_schema_employees_postgresql.png)

Aquí dintre de PostgreSQL existeix l’esquema `employees`, les taules s’han creat perfecte, amb això ja quedava feta la primera part de la pràctica: la migració de MySQL a PostgreSQL.

## Instal·lació de SymmetricDS

La segona part de la pràctica consistia a configurar SymmetricDS per replicar canvis entre les bases de dades ara vaig a instal·lar Java i vaig descarregar SymmetricDS a la màquina PostgreSQL:

```bash
sudo apt install -y openjdk-17-jre unzip curl
mkdir -p ~/ASGBD/replicacio
cd ~/ASGBD/replicacio
curl -L -o symmetricds.zip https://sourceforge.net/projects/symmetricds/files/latest/download
unzip symmetricds.zip
```

Després vaig comprovar que SymmetricDS arrenca:

```bash
java -version
cd ~/ASGBD/replicacio/symmetric-server-3.17.5
./bin/sym -v
```

![SymmetricDS instal·lat](capturas/07_symmetricds_instalat.png)

He instal·lat Java i SymmetricDS i arrencan sense errors i surt un missatge tipo "error" de que encara no hi havia fitxers pero és perquè encara no havia creat els fitxers de configuració de MySQL i PostgreSQL.

## Connexió de PostgreSQL a MySQL

Com que SymmetricDS està instal·lat a la màquina PostgreSQL, havia de poder connectar-se a MySQL de forma remota, a MySQL vaig configurar el servei perquè escoltés per xarxa, modificant:

```text
/etc/mysql/mysql.conf.d/mysqld.cnf
```

i deixant:

```conf
bind-address = 0.0.0.0
```

Després vaig crear un usuari per SymmetricDS:

```sql
CREATE USER IF NOT EXISTS 'sym'@'192.168.1.113' IDENTIFIED BY 'sym';
GRANT ALL PRIVILEGES ON employees.* TO 'sym'@'192.168.1.113';
FLUSH PRIVILEGES;
```

A la màquina PostgreSQL vaig provar la connexió:

```bash
mysql -h 192.168.1.112 -u sym -p employees
```

I dintre vaig fer:

```sql
SELECT COUNT(*) FROM employees;
```

![Connexió MySQL des de PostgreSQL](capturas/08_connexio_mysql_des_postgresql.png)

Des de `asgbd-psql-oc` puc entrar a MySQL de `asgbd-mysql-oc` i consultar la base `employees`.

## Drivers JDBC de SymmetricDS

Em vaig trobar amb que SymmetricDS necessita drivers JDBC per poder connectar-se a MySQL i PostgreSQL i vaig tindre que descarregarme els drivers:

```bash
cd ~/ASGBD/replicacio/symmetric-server-3.17.5

wget -P web/WEB-INF/lib https://repo1.maven.org/maven2/com/mysql/mysql-connector-j/8.4.0/mysql-connector-j-8.4.0.jar
wget -P web/WEB-INF/lib https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

![Drivers JDBC](capturas/09_drivers_jdbc_symmetricds.png)

Aqui estan els drivers JDBC de MySQL i PostgreSQL.

## Fitxers de configuració de SymmetricDS

SymmetricDS fa servir fitxers `.properties` dintre de la carpeta `engines`, jo he creat dos fitxers:

```text
engines/mysql.properties
engines/psql.properties
```

El fitxer de MySQL és el que configura la part de MySQL dintre de SymmetricDS:

```text
engine.name=mysql
group.id=mysql
external.id=001
sync.url=http://192.168.1.113:31415/sync/mysql
db.url=jdbc:mysql://192.168.1.112:3306/employees
db.user=sym
```

En el fitxer de PostgreSQL he posat les dades perquè SymmetricDS es pugui connectar a la base `employees` de PostgreSQL:

```text
engine.name=psql
group.id=psql
external.id=002
sync.url=http://192.168.1.113:31415/sync/psql
registration.url=http://192.168.1.113:31415/sync/mysql
db.url=jdbc:postgresql://127.0.0.1:5432/employees
db.user=postgres
```

![Fitxers engines SymmetricDS](capturas/10_fitxers_engines_symmetricds.png)

SymmetricDS necessita un fitxer de configuració per cada base de dades que ha de sincronitzar. Per això he creat dos fitxers i aqui es poden veure

## Creació de les taules internes de SymmetricDS

Després vaig crear les taules internes de SymmetricDS a MySQL i a PostgreSQL.

```bash
./bin/symadmin --engine mysql create-sym-tables
./bin/symadmin --engine psql create-sym-tables
```

Al principi vaig tenir problemes amb MySQL perquè SymmetricDS intentava crear funcions i MySQL bloquejava l’operació, per solucionar-ho vaig activar:

```sql
SET GLOBAL log_bin_trust_function_creators = 1;
```

```conf
log_bin_trust_function_creators=1
```

![Taules internes SymmetricDS](capturas/11_taules_symmetricds_creades.png)

SymmetricDS ha creat 46 taules internes a MySQL i a PostgreSQL.

## Configuració de la replicació MySQL a PostgreSQL

Per fer la replicació principal-secundari, vaig carregar la configuració a MySQL, el objectius és que MySQL sigui el node principal i PostgreSQL el node secundari, per aixo vaig crear un fitxer `mysql_to_psql.sql` amb:

```text
grups de nodes
enllaç mysql -> psql
router mysql_2_psql
triggers de les taules employees
trigger_router
```

Després el vaig carregar:

```bash
mysql -h 192.168.1.112 -u sym -p employees < mysql_to_psql.sql
```

![Configuració MySQL a PostgreSQL](capturas/12_configuracio_mysql_psql_carregada.png)

Es pot veure que existeix el grup `mysql`, el grup `psql`, el router `mysql_2_psql` i els triggers configurats per les taules de `employees`.

## Creació dels triggers a MySQL

Després vaig permetre que la part de PostgreSQL es connectés amb SymmetricDS i vaig sincronitzar els triggers:

```bash
./bin/symadmin --engine mysql open-registration psql 002
./bin/symadmin --engine mysql sync-triggers
```

Per no fer una captura enorme amb tots els triggers, vaig consultar un resum

![Triggers MySQL creats](capturas/13_triggers_mysql_creats.png)

SymmetricDS ha creat triggers a les taules principals de MySQL i els triggers són els que detecten els canvis.

## Arrencada de SymmetricDS

Vaig arrencar SymmetricDS en segon pla:

```bash
nohup ./bin/sym > logs/sym-run.log 2>&1 &
```

Per comprovar que funcionava, vaig mirar que SymmetricDS estigués en marxa, que el port `31415` estigués obert i que apareguessin MySQL i PostgreSQL com a serveis iniciats:

```bash
pgrep -af SymmetricLauncher
sudo ss -tulpn | grep :31415
grep -A4 "SymmetricDS Node STARTED" logs/sym-run.log
```

![SymmetricDS arrencat](capturas/14_symmetricds_arrencat.png)

Ara SymmetricDS està operatiu, escolta pel port `31415` i pels els nodes `mysql` i `psql` estan arrencats, durant aquesta part vaig tenir algun problema amb SymmetricDS, que necessitava el permís `PROCESS` i ho vaig apanyar amb:

```sql
GRANT PROCESS ON *.* TO 'sym'@'192.168.1.113';
FLUSH PRIVILEGES;
```

També vaig tenir un problema amb MySQL i `sym_outgoing_batch`, i s'arreglaba afegint a MySQL:

```conf
optimizer_switch=derived_merge=off
```

## Prova de replicació MySQL a PostgreSQL

Per provar que els canvis de MySQL arribaven a PostgreSQL, vaig afegir un registre nou a MySQL:

```sql
INSERT INTO departments (dept_no, dept_name)
VALUES ('d999', 'Prova Replicacio');
```

Després vaig comprovar si havia arribat a PostgreSQL:

```sql
SELECT * FROM employees.departments WHERE dept_no = 'd999';
```

![Canvi MySQL replicat a PostgreSQL](capturas/15_canvi_mysql_replicat_postgresql.png)

Podem veure que el registre `d999` apareix a PostgreSQL. Això vol dir que la replicació MySQL a PostgreSQL funciona.

## Configuració bidireccional

Per comprovar que MySQL enviava els canvis a PostgreSQL, s'ha d'afegir un registre nou a MySQL:

```text
psql -> mysql
```

i el router:

```text
psql_2_mysql
```

Carregauem la configuració i després vaig comprovem:

```sql
SELECT source_node_group_id, target_node_group_id, data_event_action
FROM sym_node_group_link;

SELECT router_id, source_node_group_id, target_node_group_id
FROM sym_router;
```

![Configuració bidireccional](capturas/16_configuracio_bidireccional.png)

Ara ja tenim els dos sentits de replicació:

```text
mysql -> psql
psql -> mysql
```

Al principi vaig carregar aquesta configuració només a MySQL, però després vaig veure que PostgreSQL no generava batches i era que PostgreSQL no tenia triggers sobre la taula `employees.departments`. Per solucionar-ho, vaig carregar també la configuració inversa dintre de PostgreSQL i després vaig executar:

```bash
./bin/symadmin --engine psql sync-triggers
```

## Prova de replicació PostgreSQL a MySQL

Per provar la replicació al contrari, vaig inserir un registre nou a PostgreSQL:

```sql
INSERT INTO employees.departments (dept_no, dept_name)
VALUES ('d997', 'Prova Bidireccional OK');
```

Després vaig comprovar a MySQL:

```sql
SELECT * FROM departments WHERE dept_no = 'd997';
```

![Canvi PostgreSQL replicat a MySQL](capturas/17_canvi_postgresql_replicat_mysql.png)

Aqui podem veure el registre creat a PostgreSQL i també apareix a MySQL, això vol dir que la replicació bidireccional esta fucionant a ala perfecció.
