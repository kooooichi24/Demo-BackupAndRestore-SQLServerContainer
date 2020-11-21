# コンテナDBのバックアップと復元のデモ
## Demo
### 1. Docker Image のプル
BookTockと同一のイメージを採用する。

```bash
$ docker pull mcr.microsoft.com/mssql/server:2017-CU21-ubuntu-16.04
```

### 2. ImageからContainerを実行(mssql1: バックアップ元のコンテナ)
volumeオプションを付与しない

```bash
$ docker run --name 'mssql1' -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=yourStrong(!)Password' -p 1433:1433 -d mcr.microsoft.com/mssql/server:2017-CU21-ubuntu-16.04
```

### 3. mssql1コンテナにデータベースとテーブル、データを作成する
### 3.1 コンテナに入る
```bash
$ docker exec -it mssql1 bash
```

### 3.2 SQLServerに接続 
```bash
$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'yourStrong(!)Password'
```

### 3.3 データベースとテーブル、データを作成
```
1> USE master;
2> GO

1> CREATE TABLE members(id int identity(1,1) primary key, name nvarchar(32));
2> GO

1> SELECT name FROM sysobjects WHERE xtype = 'U';
2> GO

1> INSERT INTO members (name) VALUES ('Koichi Furukawa');
2> GO
```

### 4. mssql1コンテナのバックアップを作成
#### 4.1 コンテナ内にバックアップを作成
```
1> BACKUP DATABASE [master] TO DISK = N'/var/opt/mssql/backup/mssql1_1.bak' WITH NOFORMAT, NOINIT, NAME = 'mssql1-ful', SKIP, NOREWIND, NOUNLOAD, STATS = 10;
2> GO
```

#### 4.2 バックアップファイルを、コンテナーからホストコンピューター上にコピー
```bash
$  docker cp mssql1:/var/opt/mssql/backup/mssql1_1.bak mssql1_1.bak
```

### 5. ImageからContainerを実行(mssql2: バックアップ先のコンテナ)
volumeオプションを付与する
```bash
$ docker-compose up -d --build
```

### 6. バックアップファイルをコンテナーにコピー
#### 6.1 docker exec を使ってバックアップ フォルダーを作成
```bash
$ docker exec -it mssql2 mkdir /var/opt/mssql/backup
```

#### 6.2 コンテナーにバックアップ ファイルをコピー
```bash
$ docker cp mssql1_1.bak mssql2:/var/opt/mssql/backup
```

### 7. データベースを復元する
#### 7.1 バックアップ内の論理ファイル名とパスを一覧表示
コンテナに入らない手法を選択します
```bash
$ docker exec -it mssql2 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'yourStrong(!)Password' -Q 'RESTORE FILELISTONLY FROM DISK = "/var/opt/mssql/backup/mssql1_1.bak"' | tr -s ' ' | cut -d ' ' -f 1-2
```

#### 7.2 コンテナー内でデータベースを復元
前の手順のファイルごとに、新しいパスを指定します。
```bash
$ docker exec -it mssql2 /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'yourStrong(!)Password' -Q 'RESTORE DATABASE master FROM DISK = "/var/opt/mssql/backup/mssql1_1.bak" WITH MOVE "master" TO "/var/opt/mssql/data/master.mdf", MOVE "mastlog" TO "/var/opt/mssql/data/mastlog.ldf"'
```

### 7.3 復元されたデータベースを確認する
```bash
$ docker exec -it mssql2 bash
$ /opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P 'yourStrong(!)Password'
```

```
>1 USE master;
>2 GO

>1 SELECT * FROM members;
>2 GO
```

