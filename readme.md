# phpmyadmin導入メモ

```yml
services:
  phpmyadmin:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: 172.19.0.1
      PMA_USER: root
      PMA_PASSWORD: <your_db_password> 
    ports:
      - "8080:80"

volumes:
  db_data:
```

```bash
$ cd /etc/mysql/mariadb.conf.d/
$ sudo vi 50-server.cnf 
###
bind-address            = 0.0.0.0
###
$ sudo systemctl restart mariadb.service
```



- とりあえず、上記で`localhost:8080`にアクセスするとphpmyadminが立ち上がることは確認できるが、ブラウザに以下のエラーが表示される。

> mysqli::real_connect(): (HY000/1130): Host '172.19.0.2' is not allowed to connect to this MariaDB server

> MySQL サーバに接続しようとしましたが拒否されました。config.inc.php のホスト、ユーザ名、パスワードが MySQL サーバの管理者から与えられた情報と一致するか確認してください。


- おそらく、docker内のアドレスが`172.19.0.1`から`localhost(127.0.0.1)`上のmariaDBを操作する権限が設定されていないことが原因の模様。

```bash
mysql -u root -p
```

```sql
MariaDB [(none)]> select user, host from mysql.user;
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| mariadb.sys | localhost |
| mysql       | localhost |
| root        | localhost |
+-------------+-----------+

```

- 従ってとりあえず（あまり良くはないが）以下のように権限を設定

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' identified by '<db_password>';
MariaDB [(none)]> select user, host from mysql.user;
+-------------+-----------+
| User        | Host      |
+-------------+-----------+
| root        | %         |
| mariadb.sys | localhost |
| mysql       | localhost |
| root        | localhost |
+-------------+-----------+

```

```bash
docker compose up -d && docker compose logs -f
```

これでとりあえずOK。
