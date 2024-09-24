# DB daily and incremental backup with S3 

```sh
#!/bin/sh

date=`date +"%d%m%Y"`
LIVE=/data/live
tm=`date +"%d%m%Y-%H_%M_%S"`
yesterday=`date --date="yesterday" +"%d%m%Y"`
BACKUP_REPO=/data/borg_backup
xtrabackup=`which xtrabackup`
borg=`which borg`
S3_CLIENT="do-backup"
S3_BUCKET="some-bucket"
S3_ACCESS_KEY="XXXX"
S3_SECRET_KEY="XXXXXX"
MYSQL_PASS=""
BORG_PRUNE_ARGS="--keep-within=1d --keep-daily=14 --keep-weekly=4 --keep-monthly=6"



if [ -e ${LIVE}/mysql_$yesterday ]; then
  rm -rf ${LIVE}/mysql_$yesterday
  rm -rf ${LIVE}/mysql_${yesterday}_incremental
fi

backup_exists=`/usr/bin/borg list $BACKUP_REPO | grep $date`

# Create target dir
if [ ! -e $LIVE/mysql_$date ]; then
mkdir $LIVE/mysql_$date

# Run xtrabackup
rm -rf $LIVE/mysql_$date/*
$xtrabackup --backup --datadir=/var/lib/mysql --target-dir=$LIVE/mysql_$date --user=root --password=$MYSQL_PASS
$xtrabackup --prepare --apply-log-only  --datadir=/var/lib/mysql --target-dir=$LIVE/mysql_$date --user=root --password=$MYSQL_PASS

exit;
fi

if [ -e $LIVE/mysql_$date ]; then

  if [ ! -e $LIVE/mysql_${date}_incremental ]; then
    mkdir $LIVE/mysql_${date}_incremental
  fi

  lastchunk=0

  for ch in `ls $LIVE/mysql_${date}_incremental | gawk  'match($0,/_([[:digit:]]+)$/,a) {print a[1]}'`; do
    if [ $ch -gt $lastchunk ]; then
    lastchunk=$ch
    fi
  done

  basedir=$LIVE/mysql_$date
  if [ $lastchunk -gt 0 ]; then
    for ch in `ls $LIVE/mysql_${date}_incremental | grep -E '*_'$lastchunk'$'`; do
    echo $ch
    basedir=$LIVE/mysql_${date}_incremental/$ch
    break
    done
  fi

  newchunk=$((lastchunk+1))
  # Run xtrabackup
  $xtrabackup --backup --incremental-basedir=$basedir --target-dir=$LIVE/mysql_${date}_incremental/${tm}_$newchunk --user=root --password=$MYSQL_PASS
fi

if [ -z $backup_exists ]; then
# Put backup into repository
    cd $LIVE/mysql_$date
    $borg create --stats --compression zlib,9 $BACKUP_REPO::$date . > /root/borg.log
    $borg prune $BORG_PRUNE_ARGS $BACKUP_REPO > /root/borg_prune.log
    RCLONE_CONFIG_DO_BACKUP_ACCESS_KEY_ID="$S3_ACCESS_KEY" RCLONE_CONFIG_DO_BACKUP_SECRET_ACCESS_KEY="$S3_SECRET_KEY" rclone sync --transfers=1 --checkers=1 $BACKUP_REPO $S3_CLIENT:$S3_BUCKET/db-backup
fi
```