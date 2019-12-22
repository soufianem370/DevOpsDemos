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



