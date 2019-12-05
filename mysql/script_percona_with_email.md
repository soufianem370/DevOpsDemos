## -------------------script de sauvegarde avec percona-------

#source 
https://www.getmysql.info/2019/05/mysql-incremental-backup-script.html

## 1)install percona extrabakup ref: https://www.percona.com/doc/percona-xtrabackup/2.4/installation/yum_repo.html
```bash
$ wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.4/\
```
binary/redhat/7/x86_64/percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm

Now you can install Percona XtraBackup by running:
```bash
$ yum localinstall percona-xtrabackup-24-2.4.4-1.el7.x86_64.rpm
```
## 2)les CMD de xtraBackup:
## backup:full==> 
```bash
$ mkdir /bkp
$ xtrabackup --user=root --password=M@roc2019 --backup --target-dir=/bkp
```
output:
xtrabackup: Transaction log of lsn (2659276) to (2659285) was copied.
191205 13:34:33 completed OK!

## restauration:===>
```bash
$ innobackupex --apply-log /bkp/
$ systemctl stop mysqld
$ rsync -avrP /bkp/ /var/lib/mysql/
$ chown -R mysql: /var/lib/mysql
$ systemctl start mysqld
```
# -----------------Installing mailx-----------------------------------------
```bash
yum -y update
yum install -y mailx
```
We can now start sending e-mails using
create a symbolic link
```bash
ln -s /bin/mailx /bin/email
```
###Set an External SMTP Server to Relay E-Mails
```bash
vi /etc/mail.rc

set smtp=smtps://smtp.gmail.com:465
set smtp-auth=login
set smtp-auth-user=soufianem370@gmail.com
set smtp-auth-password=YOURPASSWORD
set ssl-verify=ignore
set nss-config-dir=/etc/pki/nssdb/
```
==> allow using tierd application in your compte gmail
activer l'option :Accès moins sécurisé des applications in your account gmail

https://myaccount.google.com/lesssecureapps

test your server by sending an email:
```bash
echo "Your message" | mail -v -s "Message Subject" souf.makhloufi@gmail.com
```
# -----------script xtrabackup.sh-------------
```bash
#!/bin/bash
#########################################
# ###
# Features-                           ###
# FULL , Incremental, Restore ,Daily/ ###
# Weekly Backup Rotation, Mail Alert  ###
# on Successful/Failure               ###
######################################### 


SECRET="--user=root --password=M@roc2019"

## Parameters

BACKUP_DIR_ROOT=/backup/xtrabackup
BACKUP_DIR_Daily=$BACKUP_DIR_ROOT/Daily
DATA_DIR=/var/lib/mysql
HOSTIP="127.0.0.1"
SERVERNAME="nodemy11"
TAR_COMPRESS="1"  ## 0 - for disable

emails="souf.makhloufi@gmail.com" ## Add Multiple emails with space

usage() { 
echo "usage: $(basename $0) [option]" 
echo "option=full: Perform Full Backup"
echo "option=incremental: Perform Incremental Backup"
echo "option=restore: Start to Restore! Be Careful!! "
echo "option=help: show this help"
}

fullbackup() {
local TMPFILE=/tmp/XtraBackup$$.tmp
if [ ! -d $BACKUP_DIR_ROOT ]
then
mkdir $BACKUP_DIR_ROOT
fi
if [ ! -d $BACKUP_DIR_Daily ]
then
mkdir $BACKUP_DIR_Daily
fi

rm -rf $BACKUP_DIR_Daily/*

echo "$(date +%d%m%y" ""%T.%3N") :: Cleanup the backup folder is done! Starting backup" >> $BACKUP_DIR_Daily/xtrabackup_full.log

#xtrabackup --backup $SECRET -u root -p --history --compress --slave-info --compress-threads=4 --target-dir=$BACKUP_DIR_Daily/FULL

xtrabackup --backup $SECRET --history  --target-dir=$BACKUP_DIR_Daily/FULL > $TMPFILE 2>&1
#xtrabackup --backup --user=root --password=Mysqlroot@123 --history --parallel=8 --compress --compress-threads=8 --target-dir=/tmp/FULL /tmp/log 2>&1

if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then
echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_FULL failed" >> $BACKUP_DIR_Daily/xtrabackup_full.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_full.log
#echo -e $ERRMSG | mail -r "root@alert.com" -s "XtraBackup failed" $emails
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_FULL failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
rm -rf $BACKUP_DIR_Daily/FULL
exit 1
fi
rm -rf $TMPFILE
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_FULL Done!" >> $BACKUP_DIR_Daily/xtrabackup_full.log
echo -e "XtraBackup_FULL Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_FULL Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 
}

incremental_backup(){
local  TMPFILE=/tmp/XtraBackup_inc$$.tmp
if [ ! -d $BACKUP_DIR_Daily/FULL ]
then
echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:Unable to find the FULL Backup. aborting.....!" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
exit -1
fi

if [ ! -f $BACKUP_DIR_Daily/last_incremental_number ]; then
NUMBER=1
else
NUMBER=$(($(cat $BACKUP_DIR_Daily/last_incremental_number) + 1))
fi
echo "$(date +%d%m%y" ""%T.%3N") :: Starting Incremental backup inc$NUMBER" >> $BACKUP_DIR_Daily/xtrabackup_inc.log

if [ $NUMBER -eq 1 ]
then
xtrabackup --backup $SECRET --history  --incremental --target-dir=$BACKUP_DIR_Daily/inc$NUMBER --incremental-basedir=$BACKUP_DIR_Daily/FULL > $TMPFILE 2>&1
if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then

echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_inc$NUMBER failed" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_inc$NUMBER failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
rm -rf $BACKUP_DIR_Daily/inc$NUMBER
exit 1
else
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_inc$NUMBER Done!" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
echo -e "XtraBackup_inc$NUMBER Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_inc$NUMBER Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 

fi
rm -rf $TMPFILE

else
xtrabackup --backup $SECRET --history --incremental --target-dir=$BACKUP_DIR_Daily/inc$NUMBER --incremental-basedir=$BACKUP_DIR_Daily/inc$(($NUMBER - 1)) > $TMPFILE 2>&1

if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then

echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_inc$NUMBER failed" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_inc$NUMBER failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
rm -rf $BACKUP_DIR_Daily/inc$NUMBER
exit 1
else
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_inc$NUMBER Done!" >> $BACKUP_DIR_Daily/xtrabackup_inc.log
echo -e "XtraBackup_inc$NUMBER Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_inc$NUMBER Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 

fi
rm -rf $TMPFILE

fi

echo $NUMBER > $BACKUP_DIR_Daily/last_incremental_number

}

restore()
{
local  TMPFILE=/tmp/XtraBackup_restore$$.tmp
timestamp=$(date +%Y%m%d_%H%M%S)
# echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing the FULL backup" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
# xtrabackup $SECRET --decompress --remove-original --parallel=4 --target-dir=$BACKUP_DIR_Daily/FULL 
# echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing Done !!!" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: Prepareing FULL Backup ..." >> $BACKUP_DIR_Daily/xtrabackup_restore.log

xtrabackup $SECRET --prepare  --apply-log-only --target-dir=$BACKUP_DIR_Daily/FULL > $TMPFILE 2>&1

if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then

echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_FULL_Prepare failed" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_FULL_Prepare failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
exit 1
else
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_FULL_Preparation Done!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo -e "XtraBackup_FULL_Preparation Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_FULL_Preparation Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 

fi
rm -rf $TMPFILE        
if [ ! -f $BACKUP_DIR_Daily/last_prepare_number ]; then
P=1
else
P=$(cat $BACKUP_DIR_Daily/last_prepare_number)
fi
#P=1
while [ -d $BACKUP_DIR_Daily/inc$P ] && [ -d $BACKUP_DIR_Daily/inc$(($P+1)) ]
do
  # echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing incremental:$P" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
  # xtrabackup $SECRET --decompress --remove-original --parallel=4 --target-dir=$BACKUP_DIR_Daily/inc$P 
  # echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing incremental:$P Done !!!" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
  echo "$(date +%d%m%y" ""%T.%3N") :: Prepareing incremental backup inc$P" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
  xtrabackup $SECRET --prepare --apply-log-only --target-dir=$BACKUP_DIR_Daily/FULL --incremental-dir=$BACKUP_DIR_Daily/inc$P > $TMPFILE 2>&1
  
if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then

echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_inc$P Prepare failed" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_inc$P Prepare failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
exit 1
else
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_inc$P Preparation Done!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo -e "XtraBackup_inc$P preparation Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_inc$P Preparation Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 

fi
rm -rf $TMPFILE
P=$(($P+1))
echo $P > $BACKUP_DIR_Daily/last_prepare_number
done

if [ -d $BACKUP_DIR_Daily/inc$P ]
then
# echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing the last incremental:$P" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
# xtrabackup $SECRET --decompress --remove-original --parallel=4 --target-dir=$BACKUP_DIR_Daily/inc$P 
# echo `date '+%Y-%m-%d %H:%M:%S:%s'`": Decompressing the last incremental:$P Done !!!" >> $BACKUP_DIR_Daily/xtrabackup-restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: Prepareing last incremental backup inc$P" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
xtrabackup $SECRET --prepare --target-dir=$BACKUP_DIR_Daily/FULL --incremental-dir=$BACKUP_DIR_Daily/inc$P > $TMPFILE 2>&1

if [ -z "`tail -1 $TMPFILE | grep 'completed OK!'`" ] ; then

echo "$(date +%d%m%y" ""%T.%3N") :: ERROR:XtraBackup_last_inc$P Prepare failed" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: Sending Mail Alert to $emails for Backup failed in execution time" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
mail -r "soufianem370@gmail.com" -s "ERROR:XtraBackup_last_inc$P Prepare failed on $SERVERNAME@$HOSTIP at $(date)" $emails < $TMPFILE
rm -rf $TMPFILE
exit 1
else
echo "$(date +%d%m%y" ""%T.%3N") :: XtraBackup_last_inc$P Preparation Done!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo -e "XtraBackup_last_inc$P Preparation Successful! at $(date)" | mail -r "root@alert.com" -s "OK:XtraBackup_last_inc$P Preparation Successful! on $SERVERNAME@$HOSTIP at $(date)" $emails 

fi
rm -rf $TMPFILE

echo $P > $BACKUP_DIR_Daily/last_prepare_number 

fi
if [ ! -d $BACKUP_DIR_ROOT/Weekly ]
then
mkdir $BACKUP_DIR_ROOT/Weekly
fi
destdir="$BACKUP_DIR_ROOT/Weekly/FULLBACKUP_till_$timestamp"
mv $BACKUP_DIR_Daily $destdir

#### Compress........

if [ "$TAR_COMPRESS" == "1" ]; then
echo "$(date +%d%m%y" ""%T.%3N") :: $destdir compression Enabled!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
echo "$(date +%d%m%y" ""%T.%3N") :: $destdir compression Starting!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
#cd $BACKUP_DIR/$NOWDAY && sudo tar --create --gzip --file=$db.sql.tgz $db.sql --remove-files 
tar --create --gzip --file=$destdir.tgz $destdir --remove-files 
echo "$(date +%d%m%y" ""%T.%3N") :: $destdir compression Done!" >> $BACKUP_DIR_Daily/xtrabackup_restore.log
fi

}

if [ $# -eq 0 ]
then
usage
exit 1
fi

case $1 in
"full")
fullbackup
;;
"incremental")
incremental_backup
;;
"restore")
restore
;;
"help")
usage

;;
*) echo "invalid option";;
esac
    

##########################################################
### Copyright@  ###
##########################################################
```
## How to configure the cron job for automation of this script ?

```bash
$ crontab -e

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed

##This CRON will take Daily FULL backup at 12:01 AM and then incremental on every 2 Hour . Before taking the full backup it will remove all the previous backups. You can change the remove policy according to your disk space.
## Herein I'm not automate restore command . I will perform this task manually once needed. You can do as per your policy.

01 00 * * *  cd /root && ./xtrabackup.sh full    
01 */2 * * * cd /root && ./xtrabackup.sh incremental
```
