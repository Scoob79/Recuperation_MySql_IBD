# Recuperation_MySql_IBD
Procédure pour restaurer une base Mysql depuis les fichier FRM et IBD


Restauration MySQL depuis les fichiers .IBD :
=============================================

ATTENTION : ne restaurer que vos BASE pas les bases mysql, sys, phpmyadmin
			Assurez-vous que tout les fichiers sont en groupe et user MYSQL

Dans un premier temps, il faut récupérer les fichiers IBD de l'ancien serveur et les stocker sur le nouveau.
Il se trouve dans /var/lib/mysql.

Ensuite il faut générer les requêtes SQL CREATE TABLE :

La commande de base est :
	
	mysqlfrm --server={user}:{password}@localhost:3306 /sauvegarde/applications.frm --port=3310

Elle va extraire la requête pour cette base seulement.

Celle-là le fera pour tout les FRM contenu dans le dossier et sous dossier :

	for line in $(find . -iname '*.frm'); do mysqlfrm --server={user}:{password}@localhost:3306 $line; done > create.sql

Un peu de nettoyage :

	grep -V "#" create.sql > create2.sql # Vire tout les commantaires

Il vous faudra ensuite mettre ";" à la fin de toutes les requêtes.

Assurez-vous que les DATABASE soient bien toute créées sur le serveur et executez cette commande :

	mysql -u root -p < create2.sql # Envoi toutes les requêtes de création de table au serveur
	
	Si vous êtes arrivé à obtenir un fichier propre (une requête par ligne et un ";" à la fin de chaque ligne") vous pouvez utiliser cette commande :
	
		cat /media/hd/sauvegarde/sauvegarde_cerbere/frm_creation_structure_base4.sql | while  read ligne; do echo $ligne > cat; mysql -u {user} --password={password} < cat; done;

La commande {find . -iname '*.frm' | cut -d "/" -f3 | sed -e "s/.frm//g"}, executée dans le répertoire de sauvegarde devrait vous fournir le nom de toutes les tables

Il supprimer tout les nouveaux IBD du serveur et les remplacer par les anciens

Créez ce script :
	
	#!/bin/bash
	
	for line in $(find . -iname '*.frm')
	do
			base=$(echo $line | cut -d "/" -f2)
			table=$(echo $line | cut -d "/" -f3 | sed -e "s/.frm//g")
			echo "ALTER TABLE \`$base\`.\`$table\` DISCARD TABLESPACE;"
	done

Executez le dans le dossier de sauvegarde il créra les requêtes tout seul

	script.sh > req.sql
	
Cette commande executera ligne par ligne les requêtes ainsi si il y a une erreur sur une requête il ne s'arrêtera pas 

	cat req.sql | while  read ligne; do echo $ligne > cat; mysql -u {user} --password={password} < cat; done;

Copier tout les fichiers IBD de la sauvegarde vers /var/lib/mysql en respectant l'arborecense

Et pour finir importez les données :

	sed -i "s/DISCARD/IMPORT/g" req.sql
	cat req.sql | while read ligne; do echo $ligne > cat; mysql -u {user} --password={password} < cat; done;
