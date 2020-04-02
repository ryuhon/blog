---
layout: default
title: Mariabackup으로 증분백업하기 
nav_order: 3
---

# Mariabackup으로 증분백업하기 


## 전체백업
```bash
mariabackup --backup --target-dir=/backup/전체백업폴더  --user=유저아이디  --password='패스워드'  --host=localhost --slave-info 
```
```bash
mariabackup --prepare --target-dir=/backup/전체백업폴더
```
## 증분 백업후 전체 백업에 합치기 
```bash
mariabackup --backup --target-dir=/backup/증분백업폴더 --incremental-basedir=/backup/전체백업폴더  --user=유저아이디  --password='패스워드'  --host=localhost --slave-info 
```
```bash
mariabackup --prepare --target-dir=/backup/전체백업폴더  --incremental-dir=/backup/증분백업폴더
```
```bash
rm -rf /backup/증분백업폴더 
```
## 데이터 복구
```bash
mariabackup --copy-back --target-dir=/backup/전체백업폴더
chown -R mysql.mysql 데이터폴더
```

## 실행
```
systemctl start mysqld
```
