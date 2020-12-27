# 1. PPT 샤딩

- 1-0. docker 삭제

    ```bash
    docker rm -f `docker ps -aq`
    ```

- 1-1. docker run

    ```bash
    docker run -d \
      --restart=always \
      -p 3306:3306 \
      -e MYSQL_ROOT_PASSWORD=sql \
      --name=spider_db \
      mariadb:10.1

    docker run -d \
      --restart=always \
      -p 3307:3306 \
      -e MYSQL_ROOT_PASSWORD=sql \
      --name=shard_db1 \
      mariadb:10.1

    docker run -d \
      --restart=always \
      -p 3308:3306 \
      -e MYSQL_ROOT_PASSWORD=sql \
      --name=shard_db2 \
      mariadb:10.1
    ```

- 1-2. docker inspect

    ```bash
    docker inspect spider_db | grep -i ipa
    > 172.17.0.2

    docker inspect shard_db1 | grep -i ipa
    > 172.17.0.3

    docker inspect shard_db2 | grep -i ipa
    > 172.17.0.4
    ```

- 1-3. spider_db에 spider engines 추가

    ```sql
    docker exec -it spider_db bash
    find / -name install_spider.sql

    mysql -u root -p < /usr/share/mysql/install_spider.sql
    > sql
    ```

    ```bash
    mysql -u root -p
    > sql

    show engines\G
    *************************** 1. row ***************************
          Engine: SPIDER
         Support: YES
         Comment: Spider storage engine
    Transactions: YES
              XA: YES
      Savepoints: NO
    ```

- 1-4. spider_db에서 create server shard_db

    ```sql
    # spider_db

    CREATE SERVER shard_db1
     FOREIGN DATA WRAPPER mysql
    OPTIONS(
     HOST '172.17.0.3',
     DATABASE 'test',
     USER 'spider',
     PASSWORD '12121212',
     PORT 3306
    );
    ```

    ```sql
    # spider_db

    CREATE SERVER shard_db2
     FOREIGN DATA WRAPPER mysql
    OPTIONS(
     HOST '172.17.0.4',
     DATABASE 'test',
     USER 'spider',
     PASSWORD '12121212',
     PORT 3306
    );
    ```

    ```sql
    # spider_db

    select * from mysql.servers;
    +-------------+------------+------+----------+----------+------+--------+---------+-------+
    | Server_name | Host       | Db   | Username | Password | Port | Socket | Wrapper | Owner |
    +-------------+------------+------+----------+----------+------+--------+---------+-------+
    | shard_db1   | 172.17.0.3 | test | spider   | 12121212 | 3306 |        | mysql   |       |
    | shard_db2   | 172.17.0.4 | test | spider   | 12121212 | 3306 |        | mysql   |       |
    +-------------+------------+------+----------+----------+------+--------+---------+-------+
    ```

- 1-5. 디비 생성 및 계정 권한 설정

    ```sql
    # spider_db

    create database test;
    create user 'spider'@'%' identified by '12121212';
    grant all on *.* to 'spider'@'%' with grant option;
    flush privileges;
    ```

- 1-6. spider_db에서 샤딩 테이블 생성

    ```sql
    # spider_db
    # 샤딩 테이블 생성

    CREATE TABLE test.shardTest
    (
     id int(10) unsigned NOT NULL AUTO_INCREMENT,
     name char(120) NOT NULL DEFAULT '',
     PRIMARY KEY (id)
    ) ENGINE=spider COMMENT='wrapper "mysql", table "shardTest"'
     PARTITION BY KEY (id)
    (
     PARTITION shard1 COMMENT = 'srv "shard_db1"',
     PARTITION shard2 COMMENT = 'srv "shard_db2"'
    );
    ```

- 1-7. shard_db1에서 계정 생성

    ```sql
    # shard_db1 접속

    docker exec -it shard_db1 bash

    mysql -u root -p
    > sql
    ```

    ```sql
    # shard_db1 계정 생성

    create database test;
    create user 'spider'@'%' identified by '12121212';
    grant all on *.* to 'spider'@'%' with grant option;
    flush privileges;
    ```

- 1-8. shard_db1에서 테이블 생성

    ```sql
    # shard_db1

    CREATE TABLE test.shardTest
    (
     id int(10) unsigned NOT NULL AUTO_INCREMENT,
     name char(120) NOT NULL DEFAULT '',
     PRIMARY KEY (id)
    ) ENGINE=innodb;
    ```

    ```sql
    # shard_db1

    use test
    show tables;
    desc shardTest;
    ```

- 1-9. shard_db2에서 계정 생성

    ```sql
    # shard_db2

    docker exec -it shard_db2 bash

    mysql -u root -p
    > sql
    ```

    ```sql
    # shard_db2

    create database test;
    create user 'spider'@'%' identified by '12121212';
    grant all on *.* to 'spider'@'%' with grant option;
    flush privileges;
    ```

- 1-10. shard_db2에서 테이블 생성

    ```sql
    # shard_db2

    CREATE TABLE test.shardTest
    (
     id int(10) unsigned NOT NULL AUTO_INCREMENT,
     name char(120) NOT NULL DEFAULT '',
     PRIMARY KEY (id)
    ) ENGINE=innodb;
    ```

    ```sql
    # shard_db2

    use test
    show tables;
    desc shardTest;
    ```

- 1-11. spider_db에서 insert

    ```sql
    # spider_db

    docker exec -it spider_db bash

    mysql -u root -p
    > sql
    ```

    ```sql
    use test;

    # 10번 실행
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 
    insert into shardTest(name) values('aaaaa'); 

    select * from shardTest;
    ```

- 1-12. spider_db에서 select

    ```sql
    # innodb는 기본키로 정렬이 돼서 저장되는 특성이 있다.
    # 자동으로 order by 한 것과 같다.

    +----+-------+
    | id | name  |
    +----+-------+
    |  1 | aaaaa |
    |  3 | aaaaa |
    |  5 | aaaaa |
    |  7 | aaaaa |
    |  9 | aaaaa |
    |  2 | aaaaa |
    |  4 | aaaaa |
    |  6 | aaaaa |
    |  8 | aaaaa |
    | 10 | aaaaa |
    +----+-------+
    ```

- 1-13. shard_db1에서 select

    ```sql
    docker exec -it shard_db1 bash

    mysql -u root -p
    > sql

    use test;
    select * from shardTest;
    ```

    ```sql
    # 홀수만 저장

    +----+-------+
    | id | name  |
    +----+-------+
    |  1 | aaaaa |
    |  3 | aaaaa |
    |  5 | aaaaa |
    |  7 | aaaaa |
    |  9 | aaaaa |
    +----+-------+
    ```

- 1-14. shard_db2에서 select

    ```sql
    docker exec -it shard_db2 bash

    mysql -u root -p
    > sql

    use test;
    select * from shardTest;
    ```

    ```sql
    # 짝수만 저장

    +----+-------+
    | id | name  |
    +----+-------+
    |  2 | aaaaa |
    |  4 | aaaaa |
    |  6 | aaaaa |
    |  8 | aaaaa |
    | 10 | aaaaa |
    +----+-------+
    ```

# 2.  employees tables 샤딩

- 2-0. apt-get update&install

    ```bash
    docker exec -it spider_db bash

    apt-get update
    apt-get install vim -y
    apt-get install wget -y
    apt install bzip2 -y
    wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2
    bzip2 -d employees_db-full-1.0.6.tar.bz2
    tar -xvf employees_db-full-1.0.6.tar
    ```

- 2-1. spider_db show engines 확인

    ```bash
    mysql -u root -p
    > sql

    show engines\G
    *************************** 1. row ***************************
          Engine: SPIDER
         Support: YES
         Comment: Spider storage engine
    Transactions: YES
              XA: YES
      Savepoints: NO
    ```

    ```sql
    # show engines\G 입력시 Engine: SPIDER이 없으면 아래 실행

    docker exec -it spider_db bash
    find / -name install_spider.sql

    mysql -u root -p < /usr/share/mysql/install_spider.sql
    > sql
    ```

- 2-2. spider_db에서 create server employees_shard_db1

    ```sql
    # spider_db에서 추가

    CREATE SERVER employees_shard_db1
     FOREIGN DATA WRAPPER mysql
    OPTIONS(
     HOST '172.17.0.3',
     DATABASE 'employees',
     USER 'junsu_spider',
     PASSWORD 'junsuhwang',
     PORT 3306
    );
    ```

    ```sql
    # spider_db

    CREATE SERVER employees_shard_db2
     FOREIGN DATA WRAPPER mysql
    OPTIONS(
     HOST '172.17.0.4',
     DATABASE 'employees',
     USER 'junsu_spider',
     PASSWORD 'junsuhwang',
     PORT 3306
    );
    ```

    ```sql
    # spider_db

    select * from mysql.servers;
    +---------------------+------------+-----------+----------+------------+------+--------+---------+-------+
    | Server_name         | Host       | Db        | Username | Password   | Port | Socket | Wrapper | Owner |
    +---------------------+------------+-----------+----------+------------+------+--------+---------+-------+
    | shard_db1           | 172.17.0.3 | test      | spider   | 12121212   | 3306 |        | mysql   |       |
    | shard_db2           | 172.17.0.4 | test      | spider   | 12121212   | 3306 |        | mysql   |       |
    | employees_shard_db1 | 172.17.0.3 | employees | spider   | junsuhwang | 3306 |        | mysql   |       |
    | employees_shard_db2 | 172.17.0.4 | employees | spider   | junsuhwang | 3306 |        | mysql   |       |
    +---------------------+------------+-----------+----------+------------+------+--------+---------+-------+

    # 삭제
    # DROP SERVER employees_shard_db1;
    # DROP SERVER employees_shard_db2;
    ```

- 2-3. 디비 생성 및 계정 권한 설정

    ```sql
    # spider_db에서

    create database employees;
    create user 'junsu_spider'@'%' identified by 'junsuhwang';
    grant all on *.* to 'junsu_spider'@'%' with grant option;
    flush privileges;
    ```

- 2-4. spider_db에서 샤딩 테이블 생성

    ```sql
    # spider_db
    # 샤딩 테이블 생성

    CREATE TABLE employees.shardEmployees
    (
      `emp_no` int(11) NOT NULL,
      `birth_date` date NOT NULL,
      `first_name` varchar(14) NOT NULL,
      `last_name` varchar(16) NOT NULL,
      `gender` enum('M','F') NOT NULL,
      `hire_date` date NOT NULL,
      PRIMARY KEY (`emp_no`)
    ) ENGINE=spider COMMENT='wrapper "mysql", table "shardEmployees"'
     PARTITION BY KEY (`emp_no`)
    (
     PARTITION shard1 COMMENT = 'srv "employees_shard_db1"',
     PARTITION shard2 COMMENT = 'srv "employees_shard_db2"'
    );
    ```

- 2-5. shard_db1에서 계정 생성

    ```sql
    # shard_db1

    docker exec -it shard_db1 bash

    mysql -u root -p
    > sql
    ```

    ```sql
    # shard_db1

    create database employees;
    create user 'junsu_spider'@'%' identified by 'junsuhwang';
    grant all on *.* to 'junsu_spider'@'%' with grant option;
    flush privileges;
    ```

- 2-6. shard_db1에서 테이블 생성

    ```sql
    # shard_db1

    CREATE TABLE employees.shardEmployees
    (
      `emp_no` int(11) NOT NULL,
      `birth_date` date NOT NULL,
      `first_name` varchar(14) NOT NULL,
      `last_name` varchar(16) NOT NULL,
      `gender` enum('M','F') NOT NULL,
      `hire_date` date NOT NULL,
      PRIMARY KEY (`emp_no`)
    ) ENGINE=innodb;
    ```

    ```sql
    # shard_db1

    use employees
    show tables;
    desc shardEmployees;
    ```

- 2-7. shard_db2에서 계정 생성

    ```sql
    # shard_db2

    docker exec -it shard_db2 bash

    mysql -u root -p
    > sql
    ```

    ```sql
    # shard_db2

    create database employees;
    create user 'junsu_spider'@'%' identified by 'junsuhwang';
    grant all on *.* to 'junsu_spider'@'%' with grant option;
    flush privileges;
    ```

- 2-8. shard_db2에서 테이블 생성

    ```sql
    # shard_db2

    CREATE TABLE employees.shardEmployees
    (
      `emp_no` int(11) NOT NULL,
      `birth_date` date NOT NULL,
      `first_name` varchar(14) NOT NULL,
      `last_name` varchar(16) NOT NULL,
      `gender` enum('M','F') NOT NULL,
      `hire_date` date NOT NULL,
      PRIMARY KEY (`emp_no`)
    ) ENGINE=innodb;
    ```

    ```sql
    # shard_db2

    use employees
    show tables;
    desc shardEmployees;
    ```

- 2-9. spider_db에서 insert employees

    ```bash
    # spider_db

    docker exec -it spider_db bash
    ```

    ```bash
    cd /employees_db/
    sed -i 's/employees/shardEmployees/' /employees_db/load_employees.dump
    ```

    ```sql
    mysql -u root -p
    > sql

    use employees;
    source load_employees.dump
    ```

- 2-10.  spider_db에서 select

    ```sql
    select count(*) from shardEmployees;
    ```

    ```sql
    # innodb는 기본키로 정렬이 돼서 저장되는 특성이 있다.

    +----------+
    | count(*) |
    +----------+
    |   300024 |
    +----------+
    ```

- 2-11. shard_db1에서 select

    ```sql
    docker exec -it shard_db1 bash

    mysql -u root -p
    > sql

    use employees;
    select count(*) from shardEmployees;
    ```

    ```sql
    +----------+
    | count(*) |
    +----------+
    |   185853 |
    +----------+
    ```

- 2-12. shard_db2에서 select

    ```sql
    docker exec -it shard_db2 bash

    mysql -u root -p
    > sql

    use employees;
    select count(*) from shardEmployees;
    ```

    ```sql
    +----------+
    | count(*) |
    +----------+
    |   114171 |
    +----------+
    ```