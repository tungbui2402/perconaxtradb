# perconaxtradb
## 1. Chuẩn bị:
- Mysql
- 1 user mysql được cấp full quyền
- Ở file /etc/mysql/mysql.conf.d/mysqld.conf:
+ Xóa # ở 2 dòng datadir, bind-address và server-id
+ Ở bind-address sửa ip về thành 0.0.0.0
+ Ở server-id sửa thành 1, còn ở máy slave để thành 2
- Cài đặt Percona XtraBackup ở cả 2 máy:
+ Tải deb package: wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb
+ dpkg file vừa tải: sudo dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb
+ Kích hoạt repository: percona-release enable-only tools release
+ Làm mới apt: sudo apt update
+ Cài đặt percona xtra: sudo apt install percona-xtrabackup-80
+ Pinning the packages: Tạo file: /etc/apt/preferences.d/00percona.pref rồi thêm vào như sau
```
Package: *
Pin: release o=Percona Development Team
Pin-Priority: 1001
```
+ Tải file link sau: https://github.com/percona/percona-xtrabackup/tree/8.0/packaging/percona/apparmor/apparmor.d
+ Di chuyển file: sudo mv usr.sbin.xtrabackup /etc/apparmor.d/
+ Cài đặt file mới: sudo apparmor_parser -r -T -W /etc/apparmor.d/usr.sbin.xtrabackup
## 2. Backup từ máy master
Chạy lệnh: ```xtrabackup --backup --user=yourDBuser --password=MaGiCdB1 --target-dir=/path/to/backupdir```
Trong đó: 
+ user và password là của user full quyền
+ target-dir là thư mục lưu trữ backup
Output hiển thị: `xtrabackup: completed OK!` là thành công
Tạo prepare: ```xtrabackup --prepare --target-dir=/path/to/backupdir```
## 3. Truyền dữ liệu từ máy master sang máy slave
B1: dùng lệnh scp để di chuyển file backupdir vừa lưu sang máy slave:
```scp /path/to/backupdir slaveuser@slaveip:/path/to/backupdir2```
B2: Dừng mysql:```sudo systemctl stop mysql```
B3: Đổi tên file datadir thành datadir_bak: ```mv /path/to/mysql/datadir /path/to/mysql/datadir_bak```
B4: moveback file backupdir2: xtrabackup --move-back --target-dir=/path/to/mysql/backupdir
B5: Sau khi moveback thành công thì cấp quyền cho nó: ```chown mysql:mysql /path/to/mysql/datadir```
## 4. Configure master
- Đăng nhập vào mysql, sau đó tạo 1 tài khoản repl@% với quyền slave: ```GRANT REPLICATION SLAVE ON *.*  TO 'repl'@'$replicaip'
IDENTIFIED BY '$replicapass';```
- Chạy lệnh test trên máy slave: ```mysql --host=Source --user=repl --password=$replicapass```
- Xem quyền: ```SHOW GRANTS\G```
## 5. Replica
- Trên máy slave, chạy lệnh: ```cat /var/lib/mysql/xtrabackup_info | grep binlog``` để hiển thị thông tin binlog
- Đăng nhập vào mysql
- Dừng slave: ```stop slave;```
- Sử dụng change master: ```CHANGE MASTER TO MASTER_HOST='ip',MASTER_USER='repl', MASTER_PASSWORD='password', MASTER_LOG_FILE='mysql-bin.00000x', MASTER_LOG_POS=  x;```
- Khởi động slave: ```START REPLICA;```
- Kiểm tra thành công hay chưa: ```SHOW REPLICA STATUS\G```
- Output:
```
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
Seconds_Behind_Master: xx
```
### 6. Compress
- Compress files: `xtrabackup --backup --compress --user=yourDBuser --password=MaGiCdB1 --target-dir=/data/backup`
- Decompress: `xtrabackup --decompress --target-dir=/data/backup/`
- Prepare: `xtrabackup --prepare --target-dir=/data/compressed/`
- Restore như trên.
