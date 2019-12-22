## pour activé la restauration a partir des fichiers binaires:
dans votre config dans my.cnf il faut ajouter les deux lignes pour activé le binary log:
```bash
bin-log=/var/log/mysql/mysql-bin.log
--------------------------------
==>pour savoir le type du moteur utilisé dans votre  table:
show create table nom-du-table;

sur le moteur de type mysiam
  --opt(intègre l'option --lock-tables)
  --lock-all-tables (pour les exports multibases)
### exemples
sauvegarder un seule base de donnée qui se nomme "fact"
mysqldump -uroot -p --opt --lock-tables fact > /backup/fact_sauvcoherent.sql
mysqldump -uroot -p --opt --lock-all-tables --all-databases > /backup/alldatabases_sauvcoherent.sql

en production il ya que les moteurs InnoDB qui sont utilisé
sur moteur InnoDB
mysqldump -uroot -p --opt --single-transaction fact > /backup/fact_sauvcoherent.sql

======sauvegarde physique a chaud=====================
session1>flush table employes;
session1>flush table commandes;
session1>lock table employes read,commandes read;
il faut ouvrire un nouveau terminal pour copier les fichiers tables si tu fait exit
les verrous lock table seront annuler
session2>cd /var/lib/mysql/nom_du_base
cp commande* employes* /backup/tables-empl-command
apres copier phisique des deux tables,il faut revenir sur la session1 et devverouller les tables
session1>unlock tables;
======sauvegarde physique a chaud pour les moteur InnoDB===================
Sauvegarde fichiers InnoDB
 Les fichiers à sauvegarder
si sur votre configuration d'instance mysql vous avez innodb_data_file_path en on ou innodb_file_per_table=on
   innodb_data_file_path: .frm, ibdata<numéro> (un seul fichier ibdata qui regroupe l'ensembles des données pour les tables)
   innodb_file_per_table: .frm, <nom_table>.ibd (pour chaque  table vous avez ibd file par table)
 Base ouverte
   Nécessite un verrou au niveau table: 
      LOCK TABLE <nom> READ;
   Possibilité de verrouiller toutes les tables
      LOCK TABLES <table1> READ, <table2> READ, ...;
   Possibilité de verrouiller toutes les tables de l’instance
      FLUSH TABLES WITH READ LOCK;
   Nécessité d’enlever le verrou après la sauvegarde: 
      UNLOCK TABLES;
Sauvegarde fichiers InnoDB
 Les étapes
 Poser un verrou global en lecture sur l’ensemble des tables
    FLUSH TABLES WITH READ LOCK
 Sauvegarde de tous les fichiers au niveau de l’OS (cp, tar, gzip, cpio, ...)
 Dévérouillage des tables
    UNLOCK TABLES
    
    example:
    --master-data=2 permet de mettre sur la sauvegarde avec le nom du fichier binlog utilisé et la position
    si vous voulez faire la restauration il faut utilisé ce fichier binlog et la position mentiener
 #mysqldump -uroot -p --single-transaction --flush-logs --master-data=2 --all-databases > /backup/fullbackup.sql
====la resauration=========================================
Dépend du type de sauvegarde
 Restauration à partir d’une sauvegarde à froid
 Restauration à partir d’un export
 Type de restauration
• Restauration FULL, base, table
• Restauration PITR - Nécessite l’utilisation des journaux binaire - Utilisation de l’outil mysqlbinlog
==> pour initier le fichier binlog:
  mysql>flush logs;
==> pour lire un fichier binlog 
mysql-binlog.000001
mysql-binlog.000002
mysql-binlog.000003 ==> sauvegarde full
mysql-binlog.000004
mysql-binlog.000005
mysql-binlog.000006 ==> drop


mysqlbinlog mysql-binlog.000004 > /tmp/bin00004.txt
1) restaurer le full
#mysql -uroot -p < /backup/fullbackup.sql
2)pour savoir sur quelle fichier binlog ne devons commencer normalement en va lire le fichier de backup et nous allons trouver le fichier binlog avec le quelle nous allons commencer à rejouer apres la restauration full
#vi /backup/fullbackup.sql
--change master to master_log_file='mysqlbinlog.000003', master_log_pos=120
restaurer les binlog jusqu'au binlog qui pose le pb ou qui contient la commande drop
3)restaurer les binlogs
#mysqlbinlog mysql-binlog.000003 mysql-binlog.000004 mysql-binlog.000005|mysql -uroot -p




