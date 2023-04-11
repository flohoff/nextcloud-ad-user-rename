
These are tools to fixup a Nextcloud which lost its users because the AD was recreated
and all users got new **ObjectGUID** fields, which is the primary key in Nextcloud.

The first step is to get csv exports of the new and the old **sAMAccountName** and **ObjectGUID**
from the Active Directory.


Preparing a guidmap.json
========================

Then you need to create a mapping json:

	./genguidmap --guidoldcsv 20230403-flo-guidmapping-old-ad.csv \
		--guidnewcsv 20230403-markus-guidmapping-new.csv \
		-m mapfile.json

The CSV files need to contain at least the columns **sAMAcountname** and **ObjectGUID** and need
to be seperated by semicolon ";".

	ObjectGUID;sAMAccountName
	B0F74C02-1FB5-486A-820C-CDA781C873F0;Administrator

The resulting JSON basically is a map for ObjectGUID -> ObjectGUID

	jq . mapfile.json  | head -5
	{
	  "B0F74C02-1FB5-486A-820C-CDA781C873F0": {
	    "ObjectGUID": "F0E74C01-6393-446D-A61B-4FFF0D2E894A",
	    "sAMAccountName": "Administrator"
	  },
	[ ... ]

Mysql/MariaDB Dump
==================

Create a Mysql/MariaDB dump with

	mysqldump --single-transaction \
			--no-autocommit \
			--skip-extended-insert \
			nextcloud \
		>mysqldump.orig

Then run the replacement of all GUIDs:

	./process-dump \
		--sqlinput 20230324-flo-mysqldump-nextcloud.sql \
		--sqloutput  20230324-flo-mysqldump-nextcloud.fixed.sql \
		-m guidmap.json

This will take a while depending on your SQL Dump. 6 GBytes on my machine take about 20 Minutes.

Afterwards you drop the database and load the fixed sql dump


Renaming files 
===============

Nextcloud puts users files referenced by the ObjectGUID on the Datastore. So we need to rename them with the same
guidmap. This will walk the Nextcloud Datastore recursively and rename all files containing users GUIDs.

	./fixupguiddata  \
		-m guidmap.json \
		-d /data/nextcloud \
		--rename

	
