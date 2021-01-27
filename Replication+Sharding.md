# Replication+Sharding

# Docker

- 샤딩+복제 (총 5개)
    - 1-1. 우분투 로컬에 cnf 파일 만들기 (복제)

        ```bash
        cd $HOME
        pwd
        ```

        ```bash
        # shard_master_db1

        mkdir shard_master_db1
        cd shard_master_db1
        vim config_file.cnf
        > 
        [mysqld]
        log-bin=mysql-bin
        server-id=1

        cat /home/ubuntu/shard_master_db1/config_file.cnf
        ```

        ```bash
        # shard_master_db2

        cd ..
        mkdir shard_master_db2
        cd shard_master_db2

        vim config_file.cnf
        > 
        [mysqld]
        log-bin=mysql-bin
        server-id=2

        cat /home/ubuntu/shard_master_db2/config_file.cnf
        ```

        ```bash
        # shard_slave_db1

        cd ..
        mkdir shard_slave_db1
        cd shard_slave_db1

        vim config_file.cnf
        >
        [mysqld]
        server-id=3
        read_only=1

        cat /home/ubuntu/shard_slave_db1/config_file.cnf
        ```

        ```bash
        # shard_slave_db2

        cd ..
        mkdir shard_slave_db2
        cd shard_slave_db2

        vim config_file.cnf
        >
        [mysqld]
        server-id=4
        read_only=1

        cat /home/ubuntu/shard_slave_db2/config_file.cnf
        ```

    - 1-2. docker run (총 5개)

        ```bash
        # 다 날리기
        docker rm -f `docker ps -aq`
        ```

        ```bash
        # spider_db
        docker run -d \
          --restart=always \
          -p 3306:3306 \
          -e MYSQL_ROOT_PASSWORD=sql \
          --name=spider_db \
          mariadb:10.1

        # shard_master_db1
        docker run -d \
          --restart=always \
          -p 3307:3306 \
          -e MYSQL_ROOT_PASSWORD=sql \
          -v /home/ubuntu/shard_master_db1/:/etc/mysql/conf.d \
          --name=shard_master_db1 \
          mariadb:10.1

        # shard_master_db2
        docker run -d \
          --restart=always \
          -p 3308:3306 \
          -e MYSQL_ROOT_PASSWORD=sql \
          -v /home/ubuntu/shard_master_db2/:/etc/mysql/conf.d \
          --name=shard_master_db2 \
          mariadb:10.1

        # shard_slave_db1
        docker run -d \
          --restart=always \
          -p 3309:3306 \
          -e MYSQL_ROOT_PASSWORD=sql \
          -v /home/ubuntu/shard_slave_db1/:/etc/mysql/conf.d \
          --name=shard_slave_db1 \
          mariadb:10.1

        # shard_slave_db2
        docker run -d \
          --restart=always \
          -p 3310:3306 \
          -e MYSQL_ROOT_PASSWORD=sql \
          -v /home/ubuntu/shard_slave_db2/:/etc/mysql/conf.d \
          --name=shard_slave_db2 \
          mariadb:10.1

        docker ps
        ```

        ```bash
        docker exec -it shard_master_db1 bash
        cat /etc/mysql/conf.d/config_file.cnf
        exit 

        docker exec -it shard_master_db2 bash
        cat /etc/mysql/conf.d/config_file.cnf
        exit

        docker exec -it shard_slave_db1 bash
        cat /etc/mysql/conf.d/config_file.cnf
        exit

        docker exec -it shard_slave_db2 bash
        cat /etc/mysql/conf.d/config_file.cnf
        exit
        ```

    - 1-3. docker inspect

        ```bash
        docker inspect spider_db | grep -i ipa

        docker inspect shard_master_db1 | grep -i ipa

        docker inspect shard_master_db2 | grep -i ipa

        docker inspect shard_slave_db1 | grep -i ipa

        docker inspect shard_slave_db2 | grep -i ipa

        # 172.17.0.2
        # 172.17.0.3
        # 172.17.0.4
        # 172.17.0.5
        # 172.17.0.6
        ```

    - 1-4. show engines (spider_db)

        ```sql
        # show engines\G 입력시 Engine: SPIDER이 없으면 아래 실행

        docker exec -it spider_db bash
        find / -name install_spider.sql

        mysql -u root -p < /usr/share/mysql/install_spider.sql
        > sql
        ```

        ```bash
        mysql -u root -p
        > sql

        show engines\G
        * 1. row *
              Engine: SPIDER
             Support: YES
             Comment: Spider storage engine
        Transactions: YES
                  XA: YES
          Savepoints: NO
        ```

    - 1-5. create server employees_shard_master_db1 (spider_db) (종속성 있음)

        ```sql
        # spider_db에서 추가
        # 172.17.0.3 -> shard_master_db1 ip

        CREATE SERVER employees_shard_master_db1
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
        # spider_db에서 추가
        # 172.17.0.4 -> shard_master_db2 ip

        CREATE SERVER employees_shard_master_db2
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
        # spider_db에서 확인

        select * from mysql.servers;
        +----------------------------+------------+-----------+--------------+------------+------+--------+---------+-------+
        | Server_name                | Host       | Db        | Username     | Password   | Port | Socket | Wrapper | Owner |
        +----------------------------+------------+-----------+--------------+------------+------+--------+---------+-------+
        | employees_shard_master_db1 | 172.17.0.3 | employees | junsu_spider | junsuhwang | 3306 |        | mysql   |       |
        | employees_shard_master_db2 | 172.17.0.4 | employees | junsu_spider | junsuhwang | 3306 |        | mysql   |       |
        +----------------------------+------------+-----------+--------------+------------+------+--------+---------+-------+

        # 삭제
        # DROP SERVER employees_shard_master_db1;
        # DROP SERVER employees_shard_master_db2;
        ```

    - 1-6. spider_db 디비 생성 및 계정 권한 설정 (spider_db) (종속성 있음)

        ```sql
        # spider_db에서 추가

        create database employees;
        create user 'junsu_spider'@'%' identified by 'junsuhwang';
        grant all on *.* to 'junsu_spider'@'%' with grant option;
        flush privileges;
        ```

    - 1-7. spider_db 샤딩 테이블 생성 (spider_db)

        ```sql
        # spider_db에서 샤딩 테이블 생성

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
         PARTITION shard1 COMMENT = 'srv "employees_shard_master_db1"',
         PARTITION shard2 COMMENT = 'srv "employees_shard_master_db2"'
        );
        ```

        ```bash
        # 삭제
        # TRUNCATE TABLE employees.shardEmployees;
        ```

    - 1-8. shard_master_db1에서 복제 작업 (shard_master_db1)

        ```bash
        docker exec -it shard_master_db1 bash

        cat /etc/mysql/conf.d/config_file.cnf
        >
        [mysqld]
        log-bin=mysql-bin
        server-id=1

        mysql -u root -p
        > sql
        ```

        ```bash
        # master는 1, 2 slave는 3, 4

        select @@server_id;
        +-------------+
        | @@server_id |
        +-------------+
        |           1 |
        +-------------+
        ```

        ```bash
        # 계정 만들기 id: repluserOne, pw: replpwOne
        CREATE USER 'repluserOne'@'%' IDENTIFIED BY 'replpwOne';

        # 권한 주기
        GRANT REPLICATION SLAVE ON *.* TO 'repluserOne'@'%';
        ```

        ```bash
        show master status\G
        * 1. row *
                    File: mysql-bin.000005
                Position: 643
            Binlog_Do_DB:
        Binlog_Ignore_DB:
        ```

    - 1-9. shard_slave_db1에서 복제 작업 (shard_slave_db1)

        ```bash
        docker exec -it shard_slave_db1 bash

        cat /etc/mysql/conf.d/config_file.cnf
        >
        [mysqld]
        server-id=3
        read_only=1

        mysql -u root -p
        > sql
        ```

        ```sql
        # slave는 3

        select @@server_id;
        +-------------+
        | @@server_id |
        +-------------+
        |           3 |
        +-------------+
        ```

        ```sql
        # 172.17.0.3 -> shard_master_db1 ip

        CHANGE MASTER TO 
        MASTER_HOST='172.17.0.3',
        MASTER_USER='repluserOne',
        MASTER_PASSWORD='replpwOne',
        MASTER_LOG_FILE='mysql-bin.000005',
        MASTER_LOG_POS=643;
        ```

        ```sql
        start slave;
        show slave status\G
        # 11 line이 아래와 같이 떠야 함
        Slave_IO_Running: Yes
        Slave_SQL_Running: Yes
        ```

    - 1-10. shard_master_db2에서 복제 작업 (shard_master_db2)

        ```bash
        docker exec -it shard_master_db2 bash

        cat /etc/mysql/conf.d/config_file.cnf
        >
        [mysqld]
        log-bin=mysql-bin
        server-id=2

        mysql -u root -p
        > sql
        ```

        ```bash
        # master는 1, 2 slave는 3, 4
        select @@server_id;
        +-------------+
        | @@server_id |
        +-------------+
        |           2 |
        +-------------+
        ```

        ```sql
        # 계정 만들기 id: repluserTwo, pw: replpwTwo
        CREATE USER 'repluserTwo'@'%' IDENTIFIED BY 'replpwTwo';

        # 권한 주기
        GRANT REPLICATION SLAVE ON *.* TO 'repluserTwo'@'%';

        flush privileges;
        ```

        ```bash
        # master는 4에 120
        show master status\G

        * 1. row *
                    File: mysql-bin.000005
                Position: 756
            Binlog_Do_DB:
        Binlog_Ignore_DB:
        ```

        ```bash
        # 삭제
        use mysql
        drop user repluserTwo;
        ```

    - 1-11. shard_slave_db2에서 복제 작업 (shard_slave_db2)

        ```bash
        docker exec -it shard_slave_db2 bash

        cat /etc/mysql/conf.d/config_file.cnf
        >
        [mysqld]
        server-id=4
        read_only=1

        mysql -u root -p
        > sql
        ```

        ```sql
        # slave는 3

        select @@server_id;
        +-------------+
        | @@server_id |
        +-------------+
        |           4 |
        +-------------+
        ```

        ```sql
        CHANGE MASTER TO 
        MASTER_HOST='172.17.0.4',
        MASTER_USER='repluserTwo',
        MASTER_PASSWORD='replpwTwo',
        MASTER_LOG_FILE='mysql-bin.000005',
        MASTER_LOG_POS=756;
        ```

        ```sql
        start slave;
        show slave status\G
        # 11 line이 아래와 같이 떠야 함
        Slave_IO_Running: Yes
        Slave_SQL_Running: Yes
        ```

    - 1-12. shard_master_db1에서 DB&계정 생성 (shard_master_db1)

        ```sql
        # shard_master_db1

        docker exec -it shard_master_db1 bash

        mysql -u root -p
        > sql
        ```

        ```sql
        # shard_master_db1에서

        create database employees;
        create user 'junsu_spider'@'%' identified by 'junsuhwang';
        grant all on *.* to 'junsu_spider'@'%' with grant option;
        flush privileges;
        ```

    - 1-13. shard_master_db1에서 테이블 생성 (shard_master_db1)

        ```sql
        # shard_master_db1에서

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
        # shard_master_db1

        use employees
        show tables;
        desc shardEmployees;
        ```

    - 1-14. shard_master_db2에서 DB&계정 생성 (shard_master_db2)

        ```sql
        # shard_master_db2에서

        docker exec -it shard_master_db2 bash

        mysql -u root -p
        > sql
        ```

        ```sql
        # shard_master_db2에서

        create database employees;
        create user 'junsu_spider'@'%' identified by 'junsuhwang';
        grant all on *.* to 'junsu_spider'@'%' with grant option;
        flush privileges;
        ```

    - 1-15. shard_master_db2에서 테이블 생성 (shard_master_db2)

        ```sql
        # shard_master_db2에서

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
        # shard_master_db2에서

        use employees
        show tables;
        desc shardEmployees;
        ```

    ---

    ---

    ---

    - 1-16. insert employees (spider_db)

        ```bash
        # spider_db

        docker exec -it spider_db bash
        ```

        ```bash
        apt-get update -y
        apt-get install vim -y
        apt-get install wget -y
        apt install bzip2 -y
        wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2
        bzip2 -d employees_db-full-1.0.6.tar.bz2
        tar -xvf employees_db-full-1.0.6.tar
        cd employees_db
        sed -i 's/employees/shardEmployees/' /employees_db/load_employees.dump
        ```

        ```sql
        mysql -u root -p
        > sql

        use employees;
        source load_employees.dump

        commit;
        # commit을 해야 slave에서 읽힘
        # DROP DATABASE employees;
        ```

    - 1-17. select (spider_db)

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

    - 1-18. select (shard_master_db1)

        ```sql
        docker exec -it shard_master_db1 bash

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

    - 1-19. select (shard_master_db2)

        ```sql
        docker exec -it shard_master_db2 bash

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

    - 1-20. select (shard_slave_db1)

        ```bash
        docker exec -it shard_slave_db1 bash

        mysql -u root -p
        > sql
        ```

        ```sql
        use employees
        select count(*) from employees;
        ```

    - 1-21. select (shard_slave_db2)

        ```bash
        docker exec -it shard_slave_db2 bash

        mysql -u root -p
        > sql
        ```

        ```sql
        use employees
        select count(*) from employees;
        select * from employees limit 10\G;
        ```

- 읽기/쓰기 성능 테스트
    - 2-1. ubuntu docker run

        ```jsx
        docker run -it -p 8080:8080 --name=ubuntu_new1 ubuntu:18.04
        ```

        ```bash
        apt update -y
        apt install vim -y
        apt install openjdk-8-jdk -y
        apt install libmysql-java -y
        apt install tomcat8 -y
        cd /usr/share/java
        ln -s /usr/share/java/mysql-connector-java.jar /usr/share/tomcat8/lib/

        apt-get install wget -y
        apt install bzip2 -y
        wget https://launchpad.net/test-db/employees-db-1/1.0.6/+download/employees_db-full-1.0.6.tar.bz2
        bzip2 -d employees_db-full-1.0.6.tar.bz2
        tar -xvf employees_db-full-1.0.6.tar
        cd /employees_db/
        sed -i 's/employees/shardEmployees/' /employees_db/load_employees.dump

        service tomcat8 start
        ```

        ```bash
        cd /var/lib/tomcat8/webapps/ROOT
        export JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
        ln -s /usr/share/java/mysql-connector-java.jar $JAVA_HOME/lib

        export CLASSPATH=$JAVA_HOME/lib/*:.
        export PATH=$PATH:$JAVA_HOME/bin
        ```

    - 2-2. test_db inspect

        ```bash
        docker insepct test_db | grep -i ip

        # 172.17.0.8
        ```

    - 2-3. test_db create database&table

        ```bash
        # docker exec -it test_db bash

        create database employees;

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

    - 2-4. writeOneDB.jsp code

        ```jsx
        // vim /var/lib/tomcat8/webapps/ROOT/writeOneDB.jsp
        // 172.17.0.8:3306 -> test_db ip

        // <%@page contentType="text/html;charset=euc-kr" %>
        <%@page import = "java.sql.*" %>
        <%@page import="java.io.*"%>
        <%
        	long start = System.currentTimeMillis();

        	String filePath = "/employees_db/load_employees.dump";
        	Connection conn = null; 
        	PreparedStatement pstmt = null;
        	String dbURL = "jdbc:mysql://172.17.0.8:3306/employees";
        	String id = "root";
        	String pw = "sql";
        	try { 
        		FileReader fr = new FileReader(filePath);
        		BufferedReader br = new BufferedReader(fr);
        		String line = null;
        		String insertSQL = null;

            Class.forName("com.mysql.jdbc.Driver");
        		conn = DriverManager.getConnection(dbURL, id, pw);

        		while ((line = br.readLine()) != null){
        			insertSQL = line;
        			// out.write(insertSQL);
        			pstmt = conn.prepareStatement(insertSQL);
        			pstmt.executeUpdate();		
        		}
            pstmt.close();
            long end = System.currentTimeMillis();
        		
        		out.write("Record Add Success" + "<br>");

            out.write("OneDB insert time: " + (end - start)/1000.0);

            } catch(Exception e) {
        				out.println("Record ADD Fail " + e);
            }
        %>
        ```

    - 2-5. writeFiveDB.jsp code

        ```jsx
        // vim /var/lib/tomcat8/webapps/ROOT/writeFiveDB.jsp
        // 172.17.0.2:3306 -> spider_db ip

        <%@page import = "java.sql.*" %>
        <%@page import="java.io.*"%>
        <%
        	long start = System.currentTimeMillis();

        	String filePath = "/employees_db/load_employees.dump";
        	Connection conn = null; 
        	PreparedStatement pstmt = null;
        	String dbURL = "jdbc:mysql://172.17.0.2:3306/employees";
        	String id = "root";
        	String pw = "sql";
        	try { 
        		FileReader fr = new FileReader(filePath);
        		BufferedReader br = new BufferedReader(fr);
        		String line = null;
        		String insertSQL = null;

            Class.forName("com.mysql.jdbc.Driver");
        		conn = DriverManager.getConnection(dbURL, id, pw);

        		while ((line = br.readLine()) != null){
        			insertSQL = line;
        			// out.write(insertSQL);
        			pstmt = conn.prepareStatement(insertSQL);
        			pstmt.executeUpdate();		
        		}
            pstmt.close();
            long end = System.currentTimeMillis();
        		out.write("Record Add Success" + "<br>");

            out.write("FiveDB insert time: " + (end - start)/1000.0);

            } catch(Exception e) {
        				out.println("Record ADD Fail " + e);
            }
        %>
        ```

    - 2-6. readOneDB.jsp code

        ```jsx
        // vim /var/lib/tomcat8/webapps/ROOT/readOneDB.jsp
        // 172.17.0.8:3306 -> test_db ip

        <%@page import = "java.sql.*" %>
        <%@page import="java.io.*"%>
        <%

        <html>
        <head><title>READ</title></head>
        <body>
        	READ
              <table width = "100%" border = "1">
              <tr>
        						<td>emp_no</td>
                    <td>birth_date</td>
                    <td>first_name</td>
                    <td>last_name</td>
                    <td>gender</td>
                    <td>hire_date</td>         
              </tr>
         
        	long start = System.currentTimeMillis();

        	Connection conn = null; 
        	PreparedStatement pstmt = null;
        	ResultSet rs = null;

        	String dbURL = "jdbc:mysql://172.17.0.8:3306/employees";
        	String id = "root";
        	String pw = "sql";
          String selectSQL = "SELECT * FROM shardEmployees ORDER BY RAND() LIMIT 20000;";

        	try { 
            Class.forName("com.mysql.jdbc.Driver");
        		conn = DriverManager.getConnection(dbURL, id, pw);

        		pstmt = conn.prepareStatement(selectSQL);
        		rs = pstmt.executeQuery();

        		while (rs.next()){
        %>
        			<tr>
        			      <td><%= rs.getString("emp_no") %></td>
        		        <td><%= rs.getString("birth_date") %></td>
        		        <td><%= rs.getString("first_name") %></td>
        		        <td><%= rs.getString("last_name") %></td>
        		        <td><%= rs.getString("gender") %></td>
        		        <td><%= rs.getString("hire_date") %></td>
        			</tr>
        <%

        		}
            pstmt.close();
            long end = System.currentTimeMillis();

        		out.write("OneDB select Success" + "<br>");
            out.write("OneDB select time: " + (end - start)/1000.0);

            } catch(Exception e) {
        				out.println("Search Fail " + e);
            }
        %>

              </table>
        </body>
        </html>
        ```

    - 2-7. readFiveDB.jsp code

        ```jsx
        // vim /var/lib/tomcat8/webapps/ROOT/readFiveDB.jsp
        // 172.17.0.2:3306 -> spider_db ip

        <%@page import = "java.sql.*" %>
        <%@page import="java.io.*"%>
        <%

        <html>
        <head><title>READ</title></head>
        <body>
        	READ
              <table width = "100%" border = "1">
              <tr>
        						<td>emp_no</td>
                    <td>birth_date</td>
                    <td>first_name</td>
                    <td>last_name</td>
                    <td>gender</td>
                    <td>hire_date</td>         
              </tr>
         
        	long start = System.currentTimeMillis();

        	Connection conn = null; 
        	PreparedStatement pstmt = null;
        	ResultSet rs = null;

        	String dbURL = "jdbc:mysql://172.17.0.2:3306/employees";
        	String id = "root";
        	String pw = "sql";
          String selectSQL = "SELECT * FROM shardEmployees ORDER BY RAND() LIMIT 20000;";

        	try { 
            Class.forName("com.mysql.jdbc.Driver");
        		conn = DriverManager.getConnection(dbURL, id, pw);

        		pstmt = conn.prepareStatement(selectSQL);
        		rs = pstmt.executeQuery();

        		while (rs.next()){
        %>
        			<tr>
        			      <td><%= rs.getString("emp_no") %></td>
        		        <td><%= rs.getString("birth_date") %></td>
        		        <td><%= rs.getString("first_name") %></td>
        		        <td><%= rs.getString("last_name") %></td>
        		        <td><%= rs.getString("gender") %></td>
        		        <td><%= rs.getString("hire_date") %></td>
        			</tr>
        <%

        		}
            pstmt.close();
            long end = System.currentTimeMillis();

        		out.write("FiveDB select Success" + "<br>");
            out.write("FiveDB select time: " + (end - start)/1000.0);

            } catch(Exception e) {
        				out.println("Search Fail " + e);
            }
        %>

              </table>
        </body>
        </html>
        ```