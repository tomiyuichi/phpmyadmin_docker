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


# docker networkの調べ方

- phpmyadminのコンテナを作ったときにデフォルトで作られるネットワークのアドレスを調べる。

```bash
tomi@mori-Inspiron5491:~/phpmyadmin$ docker network ls
NETWORK ID     NAME                 DRIVER    SCOPE
92d585f4a2a6   bridge               bridge    local
976adbc1151b   host                 host      local
9936f33d6ab1   none                 null      local
146bf73d798f   phpmyadmin_default   bridge    local
d8727fc9112a   wetty                bridge    local
```

```bash
tomi@mori-Inspiron5491:~/phpmyadmin$ docker network inspect phpmyadmin_default 
[
    {
        "Name": "phpmyadmin_default",
        "Id": "146bf73d798f9d359070b0315830fd02f7882f9e87a723bb20bdcef6ee637d39",
        "Created": "2025-02-10T17:57:10.628305189+09:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.19.0.0/16",
                    "Gateway": "172.19.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "5424fc950ed046ba7a6aaff4e11a8bded57795e65902c759b801054518efbd60": {
                "Name": "phpmyadmin-phpmyadmin-1",
                "EndpointID": "8a41da193a72c1b4e7147a2d47d48ae25403d391cfe481ced1a7b8e790a5f164",
                "MacAddress": "02:42:ac:13:00:02",
                "IPv4Address": "172.19.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {
            "com.docker.compose.config-hash": "ba43439e637b9a85c5cc5a0e7f1026048251f55346bbe10cfb89f592f05e5f3c",
            "com.docker.compose.network": "default",
            "com.docker.compose.project": "phpmyadmin",
            "com.docker.compose.version": "2.32.4"
        }
    }
]


```

- 上記から、`localhost`に対して`172.19.0.1`が割り当てられていると判断する。
- ifconfigを実行し、`172.19.0.1`が存在することも確認する。



