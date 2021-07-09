# Intensive Final Exam

### 금융Digital 1그룹 - 정근화(10408)

정근화 - Student 12
퍼블릭 IP	프라이빗 IP
호스트1	3.37.170.239	10.0.0.114
호스트2	3.37.176.133	10.0.0.93
호스트3	3.37.176.141	10.0.0.102
호스트4	3.37.179.126	10.0.0.103
호스트5	3.37.183.172	10.0.0.104

### 0. Challenges

a. Create and place your work in challenges folder in your Github project

b. All screenshot must be in PNG format

c. All text file(including command set) require Markdown(.md) formatting.

d. you will create each required file yourself

e. Make sure you provide screenshots and/or cut and paste of your command line outputs for each of the items that you are asked todo.

f. It is ok to combine multiple requirements into a single screenshot and/or cut/paste of your terminal

### 1. Create a CDH Cluster on AWS

- Linux setup
    1. Add the following linux accounts to all nodes
        - User training with a UID of 8800
        - Set the password for user “training” to “training”
        - Create the group skcc and add training to it
        - Give training sudo capabilities

        ```sql
        // training uid 8800 유저 생성
        [centos@ip-10-0-0-114 ~]$ sudo useradd -u 8800 training

        // 패스워드 설정
        [centos@ip-10-0-0-114 ~]$ sudo passwd training
        Changing password for user training.
        New password:
        BAD PASSWORD: The password contains the user name in some form
        Retype new password:
        passwd: all authentication tokens updated successfully.

        // skcc 그룹 추가 및 training 유저 추가
        [centos@ip-10-0-0-114 ~]$ sudo groupadd skcc
        [centos@ip-10-0-0-114 ~]$ sudo gpasswd -a training skcc
        Adding user training to group skcc
        [centos@ip-10-0-0-114 ~]$ sudo vi /etc/group
        ...
        training:x:8800:
        skcc:x:8801:training

        // Sudo 권한 부여
        // 이때 Nopasswd 설정해두면 나중에 cloudera manager 에서 training 유저로 sudo 권한을 행사할 때 제약이 없어서 편함.
        [centos@ip-10-0-0-114 ~]$ sudo chmod 640 /etc/sudoers
        [centos@ip-10-0-0-114 ~]$ sudo vi /etc/sudoers
        training ALL=NOPASSWD: ALL
        ```

    2. List the your instances by IP address and DNS name (don’t use /etc/hosts
    for this)

        ```sql
        [centos@ip-10-0-0-114 ~]$ cat /etc/resolv.conf
        ; generated by /usr/sbin/dhclient-script
        search ap-northeast-2.compute.internal
        nameserver 10.0.0.2
        ```

    3. List the Linux release you are using

        ```sql
        [centos@ip-10-0-0-114 ~]$ cat /etc/redhat-release
        CentOS Linux release 7.9.2009 (Core)
        ```

    4. List the file system capacity for the first node (master node)

        ```sql
        [centos@ip-10-0-0-114 ~]$ df -h
        Filesystem      Size  Used Avail Use% Mounted on
        devtmpfs        7.7G     0  7.7G   0% /dev
        tmpfs           7.7G     0  7.7G   0% /dev/shm
        tmpfs           7.7G   17M  7.7G   1% /run
        tmpfs           7.7G     0  7.7G   0% /sys/fs/cgroup
        /dev/nvme0n1p1  100G  1.5G   99G   2% /
        tmpfs           1.6G     0  1.6G   0% /run/user/1000
        tmpfs           1.6G     0  1.6G   0% /run/user/0
        ```

    5. List the command and output for yum repolist enabled

        ```sql
        [centos@cm yum.repos.d]$ yum repolist all | grep enable
        base/7/x86_64                       CentOS-7 - Base              enabled: 10,072
        cloudera-manager                    Cloudera Manager             enabled:      7
        extras/7/x86_64                     CentOS-7 - Extras            enabled:    498
        updates/7/x86_64                    CentOS-7 - Updates           enabled:  2,459
        ```

    6. List the /etc/passwd entries for training (only in master name node)

        ```sql
        [centos@ip-10-0-0-114 ~]$ sudo cat /etc/passwd | grep training
        training:x:8800:8800::/home/training:/bin/bash
        ```

    7. List the /etc/group entries for skcc (only in master name node)

        ```sql
        [centos@ip-10-0-0-114 ~]$ sudo cat /etc/group | grep skcc
        skcc:x:8801:training
        ```

    8. List output of the flowing commands:
        1. getent group skcc
        2. getent passwd training

        ```sql
        [centos@ip-10-0-0-114 ~]$ getent group skcc
        skcc:x:8801:training
        [centos@ip-10-0-0-114 ~]$ getent passwd training
        training:x:8800:8800::/home/training:/bin/bash
        ```

- 각 리눅스 노드 추가 선행 세팅

    ```sql
    // ssh 접속 패스워드로 가능하게 세팅
    [centos@ip-10-0-0-114 ~]$ sudo vi /etc/ssh/sshd_config
    ...
    PasswordAuthentication yes
    ...
    [centos@ip-10-0-0-114 ~]$ sudo systemctl restart sshd.service

    // host 세팅 및 현재 host 이름 지정
    [centos@ip-10-0-0-114 ~]$ sudo vi /etc/hosts
    ...
    10.0.0.114      cm.skcc.com     cm
    10.0.0.93       m1.skcc.com     m1
    10.0.0.102      d1.skcc.com     d1
    10.0.0.103      d2.skcc.com     d2
    10.0.0.104      d3.skcc.com     d3
    [centos@ip-10-0-0-114 ~]$ sudo hostnamectl set-hostname cm.skcc.com
    [centos@ip-10-0-0-114 ~]$ hostname -f
    cm.skcc.com

    // jdk1.8 설치
    [centos@ip-10-0-0-114 ~]$ sudo yum install java-1.8.0-openjdk-devel

    // mariadb 접속을 위한 jdbc 설치 -> yum 패키지에 mysql connector가 바로 있지 않아서
    // tar 압축파일 가져온다음에 지정 경로에 압축해제하는 방식으로 jdbc 라이브러리 설치
    [centos@ip-10-0-0-114 ~]$ sudo yum install -y wget
    [centos@ip-10-0-0-114 ~]$ sudo wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.47.tar.gz ~/
    [centos@ip-10-0-0-114 ~]$ sudo tar zxvf mysql-connector-java-5.1.47.tar.gz
    [centos@ip-10-0-0-114 ~]$ sudo mkdir -p /usr/share/java/
    [centos@ip-10-0-0-114 ~]$ sudo cp ~/mysql-connector-java-5.1.47/mysql-connector-java-5.1.47-bin.jar /usr/share/java/mysql-connector-java.jar

    // 모두 세팅하고 한번 재부팅
    [centos@ip-10-0-0-114 ~]$ sudo init 6
    ```

- Install a MySQL server
    - 선행 세팅 : CM 노드에 Yum 레포지토리 추가 및 cloudera 매니징 서버 설치

    ```sql
    // 선행 세팅 - cloudera 관련 server 및 agent 를 설치하기 위해서는 yum 에 cloudera 레포지토리를 추가해주어야 한다.
    [centos@cm ~]$ sudo wget http://ec2-3-34-114-205.ap-northeast2.compute.amazonaws.com/cloudera-repos/cdh5/5.16.2/cloudera-manager.repo
    [centos@cm yum.repos.d]$ sudo vi cloudera-manager.repo
    [centos@cm yum.repos.d]$ yum repolist all | grep enable
    base/7/x86_64                       CentOS-7 - Base              enabled: 10,072
    cloudera-manager                    Cloudera Manager             enabled:      7
    extras/7/x86_64                     CentOS-7 - Extras            enabled:    498
    updates/7/x86_64                    CentOS-7 - Updates           enabled:  2,459

    [centos@cm yum.repos.d]$ sudo yum install -y cloudera-manager-daemons cloudera-manager-server
    ```

    1. Use MariaDB as the database for all the services. You may choose your own username and passwords but make a record of it so that we may access them.

        ```sql
        // mariadb 서버 설치 실행, 보안 세팅
        [centos@cm yum.repos.d]$ sudo yum install -y mariadb-server
        [centos@cm yum.repos.d]$ sudo systemctl enable mariadb
        Created symlink from /etc/systemd/system/multi-user.target.wants/mariadb.service to /usr/lib/systemd/system/mariadb.service.
        [centos@cm yum.repos.d]$ sudo systemctl start mariadb
        [centos@cm yum.repos.d]$ sudo /usr/bin/mysql_secure_installation

        // mariadb 접속
        [centos@cm yum.repos.d]$ mysql -u root -p

        // 각 hadoop 시스템에서 사용할 DB 생성
        MariaDB [(none)]> CREATE DATABASE scm DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON scm.* TO 'scm-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE amon DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON amon.* TO 'amon-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE rman DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.01 sec)

        MariaDB [(none)]> GRANT ALL ON rman.* TO 'rman-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE hue DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON hue.* TO 'hue-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE metastore DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON metastore.* TO 'metastore-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE sentry DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON sentry.* TO 'sentry-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> CREATE DATABASE oozie DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci;
        Query OK, 1 row affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON oozie.* TO 'oozie-user'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> GRANT ALL ON test.* TO 'eduuser'@'%' IDENTIFIED BY 'training';
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> flush privileges;
        Query OK, 0 rows affected (0.00 sec)

        MariaDB [(none)]> show databases;
        +--------------------+
        | Database           |
        +--------------------+
        | information_schema |
        | amon               |
        | hue                |
        | metastore          |
        | mysql              |
        | oozie              |
        | performance_schema |
        | rman               |
        | scm                |
        | sentry             |
        +--------------------+
        10 rows in set (0.00 sec)

        // scm 세팅 및 서버 실행
        [centos@cm yum.repos.d]$ sudo /usr/share/cmf/schema/scm_prepare_database.sh mysql scm scm-user training
        JAVA_HOME=/usr/lib/jvm/java-openjdk
        Verifying that we can write to /etc/cloudera-scm-server
        Creating SCM configuration file in /etc/cloudera-scm-server
        Executing:  /usr/lib/jvm/java-openjdk/bin/java -cp /usr/share/java/mysql-connector-java.jar:/usr/share/java/oracle-connector-java.jar:/usr/share/java/postgresql-connector-java.jar:/usr/share/cmf/schema/../lib/* com.cloudera.enterprise.dbutil.DbCommandExecutor /etc/cloudera-scm-server/db.properties com.cloudera.cmf.db.
        [                          main] DbCommandExecutor              INFO  Successfully connected to database.
        All done, your SCM database is configured correctly!
        [centos@cm yum.repos.d]$ sudo systemctl start cloudera-scm-server

        ```

    2. List the following in your GitHub
        - A command and output that shows the hostname of your database server

            ```sql
            [centos@cm yum.repos.d]$ hostname -f
            cm.skcc.com
            ```

        - A command and output that reports the database server version

            ```sql
            [centos@cm yum.repos.d]$ mysql -u root -p

            MariaDB [(none)]> select version();
            +----------------+
            | version()      |
            +----------------+
            | 5.5.68-MariaDB |
            +----------------+
            1 row in set (0.00 sec)
            ```

        - A command and output that lists all the databases in the server

            ```sql
            [centos@cm yum.repos.d]$ mysql -u root -p

            MariaDB [(none)]> show databases;
            +--------------------+
            | Database           |
            +--------------------+
            | information_schema |
            | amon               |
            | hue                |
            | metastore          |
            | mysql              |
            | oozie              |
            | performance_schema |
            | rman               |
            | scm                |
            | sentry             |
            +--------------------+
            10 rows in set (0.00 sec)
            ```

- Install Cloudera Manage

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.43.33.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.43.33.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.47.45.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.47.45.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.49.10.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_14.49.10.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.10.38.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.10.38.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.18.28.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.18.28.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.27.38.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.27.38.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.31.51.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.31.51.png)

    ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.32.21.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.32.21.png)

    1. Specifically, you MUST install CDH version 5.16.2
    2. The Cluster does not have to be in HA mode.
    3. Make sure that the following services (and any necessary services to
    install that service) are installed:
        1. HDFS
        2. YARN
        3. Sqoop
        4. Hive
        5. Impala
        6. HUE
    4. In you cluster, create a user named “training” with password “training”
    5. You should have already created the linux user from previous
    step. Now, make sure user “training” has both a linux and HDFS
    home directory

        ![Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.38.55.png](Intensive%20Final%20Exam%2034953d083a804fa5839b7c1ffbc70d38/_2021-07-09_15.38.55.png)

- In MySQL create the sample tables that will be used for the rest of the test
    1. In MySQL, create a database and name it “test”
    2. Create 2 tables in the test databases: authors and posts.
        - You will use the authors.sql and posts.sql script files that will be provided
        for you to generate the necessary tables
    3. Create and grant user “training” with password “training” full access to the test
    database. (It is ok if you give training full access to the entire MySQL database)

- Extract tables authors and posts from the database and create Hive tables.
    1. Use Sqoop to import the data from authors and posts
    2. For both tables, you will import the data in tab delimited text format
    3. The imported data should be saved in training’s HDFS home directory
        - Create authors and posts directories in your HDFS home directory
        - Save the imported data in each
    4. In Hive, create 2 tables: authors and posts. They will contain the data that you
    imported from Sqoop in above step.
    5. You are free to use whatever database in Hive.
    6. Create authors as an external table.
    7. Create posts as a managed table.

- Create and run a Hive/Impala query. From the query, generate the results dataset that
you will use in the next step to export in MySQL.
    1. Create a query that counts the number of posts each author has created.
        - The id column in authors matches the author_id key in posts.
    2. The output of the query should provide the following information:

        Source Output Column Name
        Id from authors Id
        first_name from authors fname
        last_name from authors Lname
        Aggregated count of number of posts num_posts

    3. The output of the query should be saved in your HDFS home directory.
        - Save it under “results” directory