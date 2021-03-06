Les Foreign Data Wrappers permettent à une BDD PostgreSQL de lancer une requête locale qui interroge une source distance (BDD PostgreSQL, fichier CSV, SHP, MySQL, Access...).

Voir la documentation officielle : https://www.postgresql.org/docs/9.5/static/postgres-fdw.html

Il faut recréer localement la structure de la source distante. Dans cet exemple, on va se connecter à une BDD PostgreSQL.

Commencer par ajouter l'extension à la BDD locale (lizmapdb dans notre cas) en ligne de commande : 

::

  sudo -n -u postgres -s psql -d lizmapdb -c "CREATE EXTENSION IF NOT EXISTS postgres_fdw;"

Ou directement en SQL : 

::

  CREATE EXTENSION IF NOT EXISTS postgres_fdw;

Créez ensuite localement la connexion au serveur source (nommée geonaturedbserver dans notre cas) :

::

  CREATE SERVER geonaturedbserver FOREIGN DATA WRAPPER postgres_fdw OPTIONS (host 'IP-DB-source', dbname 'DB-source-name', port '5432');

Créez un mapping entre votre utilisateur de BDD local et l'utilisateur à utiliser au niveau du serveur source :

::

  CREATE USER MAPPING 
   FOR lizmapadmin
   SERVER geonaturedbserver
  OPTIONS (user 'user-DB-source',password 'user-password');

Créer le schéma que vous souhaitez copier dans la BDD locale

::

  CREATE SCHEMA synthese;

Depuis PostgreSQL 9.5, il est possible d'importer automatiquement la structure d'un schéma distant dans la BDD locale :

::

  IMPORT FOREIGN SCHEMA synthese
    FROM SERVER geonaturedbserver INTO synthese;

On fait pareil avec le schéma `taxonomie` :

::

  CREATE SCHEMA taxonomie;
  IMPORT FOREIGN SCHEMA taxonomie
    FROM SERVER geonaturedbserver INTO taxonomie;
	
On peut alors créer localement des vues matérialisées pour disposer des données mises en forme localement. 
Exemple de toutes observations d'un organisme : 

::

  CREATE MATERIALIZED VIEW synthese.vm_observations_pne AS 
   SELECT * FROM synthese.syntheseff s
    WHERE s.id_organisme = 2	

Les vues matérialisées permettent de stocker le résultat d'une vue et donc d'optimiser les performances en pré-calculant le résultat d'une vue et en disposant des données localement dans le cas d'utilisation de FDW.

Néanmoins il faut rafraichir ces données si la donnée source a été modifiée. Cela peut-être manuellement, vue matérialisée par vue matérialisée. 

Il est aussi possible de mettre en place une fonction qui rafraichit toutes les vues matérialisées et de lancer celle-ci régulièrement à l'aide d'un CRON comme expliqué dans la documentation de GeoNature-atlas : https://github.com/PnEcrins/GeoNature-atlas/blob/master/docs/vues_materialisees_maj.rst#mise-%C3%A0-jour-des-vues-mat%C3%A9rialis%C3%A9es


Lister les foreigns data wrapper
================================

En SQL
::
	-- Liste des FDW définis
	SELECT srvname, srvowner::regrole, w.fdwname, srvtype, srvversion, srvacl, srvoptions  
	FROM pg_foreign_server 
	JOIN pg_foreign_data_wrapper w 
	ON w.oid = srvfdw;
	
	-- Liste des mappings utilisateurs
	SELECT s.srvname, m.* 
	FROM pg_user_mappings m 
	JOIN pg_foreign_server s ON s.oid = m.srvid;

	
En psql
:: 
	\des+

docs System Catalogs :
 * https://www.postgresql.org/docs/current/static/catalog-pg-foreign-server.html
 * https://www.postgresql.org/docs/current/static/view-pg-user-mappings.html


lister l'ensemble des fdw d'un serveur
---------------------------------------

le résultat est stocké dans un fichier csv /tmp/output.csv


::

	db_user_pg=monsuperuser
	db_user_pg_pass=monpassachanger
	db_host=monhote


	echo "db_name;srvname;srvowner;fdwname;srvtype;srvversion;srvacl;srvoptions"  > /tmp/output.csv
	export PGPASSWORD=${db_user_pg_pass};psql -tl -h $db_host -U $db_user_pg | while read -a Record ; do
	    echo ${Record[0]}
		export PGPASSWORD=${db_user_pg_pass};psql -h $db_host -U $db_user_pg -d ${Record[0]} -t -A -F ";" -c "SELECT '${Record[0]}', srvname, srvowner::regrole, w.fdwname, srvtype, srvversion, srvacl, srvoptions
		FROM pg_foreign_server
		JOIN pg_foreign_data_wrapper w
		ON w.oid = srvfdw;"  >> /tmp/output.csv
	done
