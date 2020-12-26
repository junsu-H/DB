

- 1-0. docker 전부 삭제

    ```bash
    docker rm -f $(docker ps -aq)
    docker ps -a
    ```

- 1-1. 우분투 로컬에 cnf 파일 만들기

    ```bash
    vim /home/ubuntu/master/config_file.cnf
    > 
    [mysqld]
    log-bin=mysql-bin
    server-id=1

    vim /home/ubuntu/slave/config_file.cnf
    >
    [mysqld]
    server-id=2
    read_only=1
    ```

- 1-2. docker run

    ```bash
    # master
    docker run -d --restart=always -p 3306:3306 \
    -e MYSQL_ROOT_PASSWORD=ssu \
    -v /home/ubuntu/master/:/etc/mysql/conf.d \
    --name=mysql-master mysql:5.6

    # slave
    docker run -d --restart=always -p 3307:3306 \
    -e MYSQL_ROOT_PASSWORD=ssu \
    -v /home/ubuntu/slave/:/etc/mysql/conf.d \
    --name=mysql-slave mysql:5.6
    ```

- 1-3. IP Check

    ```bash
    # ip 확인
    docker inspect mysql-master | grep -i ip # 172.17.0.2
    docker inspect mysql-slave| grep -i ip # 172.17.0.3
    ```

- 1-4. docker master_info

    ```bash
    docker exec -it mysql-master bash

    cat /etc/mysql/conf.d/config_file.cnf
    [mysqld]
    log-bin=mysql-bin
    server-id=1

    mysql -u root -p
    > ssu
    ```

    ```bash
    # master는 1, slave는 2
    select @@server_id;
    +-------------+
    | @@server_id |
    +-------------+
    |           1 |
    +-------------+
    ```

- 1-5. master에서 계정 생성

    ```bash
    # 계정 만들기 id: repluser, pw: replpw
    CREATE USER 'repluser'@'%' IDENTIFIED BY 'replpw';

    # 권한 주기
    GRANT REPLICATION SLAVE ON *.* TO 'repluser'@'%';
    ```

    ```bash
    # master는 4에 120
    show master status;
    +------------------+----------+--------------+------------------+-------------------+
    | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +------------------+----------+--------------+------------------+-------------------+
    | mysql-bin.000004 |      120 |              |                  |                   |
    +------------------+----------+--------------+------------------+-------------------+
    ```

- 1-6. docker slave_info

    ```bash
    docker exec -it mysql-slave bash

    cat /etc/mysql/conf.d/config_file.cnf
    [mysqld]
    server-id=2
    read_only=1

    mysql -u root -p
    > ssu
    ```

    ```sql
    # slave는 2
    select @@server_id;
    +-------------+
    | @@server_id |
    +-------------+
    |           2 |
    +-------------+
    ```

- 1-7.  slave에서 change master to

    ```sql
    # slave에서 진행

    CHANGE MASTER TO 
    MASTER_HOST='172.17.0.2', # master ip
    MASTER_USER='repluser', # master에서 등록한 user
    MASTER_PASSWORD='replpw', # master에서 등록한 pw
    MASTER_LOG_FILE='mysql-bin.000004', # show master status\G에서 나온 값
    MASTER_LOG_POS=120; # show master status\G에서 나온 값
    ```

    ```sql
    start slave;
    show slave status\G;
    # 11 line이 아래와 같이 떠야 함
    Slave_IO_Running: Yes
    Slave_SQL_Running: Yes
    ```

- 1-8. master에서 employees install

    ```bash
    docker exec -it mysql-master bash

    apt-get update
    apt-get install vim -y
    apt-get install wget -y
    apt install bzip2 -y
    wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2
    bzip2 -d employees_db-full-1.0.6.tar.bz2
    tar xvf employees_db-full-1.0.6.tar

    cd employees_db
    mysql -u root -p
    > ssu
    ```

    ```sql
    source employees.sql

    # autocommit=0이면 commit을 해야 됨.
    # commit

    # 삭제
    # DROP DATABASE employees;
    ```

- 1-9. slave에서 복제되었는지 확인

    ```bash
    docker exec -it mysql-slave bash

    mysql -u root -p
    > ssu
    ```

    ```sql
    use employees
    select count(*) from employees;
    select * from employees limit 10\G;
    ```