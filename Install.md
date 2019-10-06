# 携程Apollo安装

## 安装必备组件

```bash
$ yum -y install java-1.8.0-openjdk
$ yum -y install mysql-server-5.7
```

## 创建apollo用户

```bash
egrep "^apollo" /etc/passwd >& /dev/null
if [ $? -ne 0 ]
then
    useradd -M -s /sbin/nologin apollo
fi
```

## 创建apollo文件夹

```bash
$ install -d /usr/local/apollo
```

## 安装adminservice

```bash
$ cd /usr/local/apollo
$ curl -o apollo-adminservice-1.4.0-github.zip https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-adminservice-1.4.0-github.zip
$ unzip apollo-adminservice-1.4.0-github.zip -d apollo-adminservice-1.4.0-github
$ ln -s apollo-adminservice-1.4.0-github adminservice
```

```bash
$ vi adminservice/config/application-github.properties
```

```conf
spring.datasource.url = jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
spring.datasource.username = apollo
spring.datasource.password = XXXXXX
```

## 安装configservice

```bash
$ cd /usr/local/apollo
$ curl -o apollo-configservice-1.4.0-github.zip https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-configservice-1.4.0-github.zip
$ unzip apollo-configservice-1.4.0-github.zip -d apollo-configservice-1.4.0
$ ln -s apollo-configservice-1.4.0-github configservice
```

```bash
$ vi configservice/config/application-github.properties
```

```conf
spring.datasource.url = jdbc:mysql://localhost:3306/ApolloConfigDB?characterEncoding=utf8
spring.datasource.username = apollo
spring.datasource.password = XXXXXX
```

## 安装portal

```bash
$ cd /usr/local/apollo
$ curl -o apollo-portal-1.4.0-github.zip https://github.com/ctripcorp/apollo/releases/download/v1.4.0/apollo-portal-1.4.0-github.zip
$ unzip apollo-portal-1.4.0-github.zip -d apollo-portal-1.4.0
$ ln -s apollo-portal-1.4.0-github portal
```

```bash
$ vi portal/config/application-github.properties
```

```conf
spring.datasource.url = jdbc:mysql://localhost:3306/ApolloPortalDB?characterEncoding=utf8
spring.datasource.username = apollo
spring.datasource.password = XXXXXX
```

```bash
$ vi configservice/config/apollo-env.properties
```

```conf
dev.meta=http://localhost:8080
```

## OpenLDAP

```bash
$ vi portal/config/application-ldap.yml
```

```yml
spring:
  ldap:
    base: "dc=domain,dc=com"
    username: "cn=root,dc=domain,dc=com" # 配置管理员账号，用于搜索、匹配用户
    password: "XXXXXX"
    searchFilter: "(uid={0})"  # 用户过滤器，登录的时候用这个过滤器来搜索用户
    urls:
    - "ldap://localhost:389"

ldap:
  mapping: # 配置 ldap 属性
    objectClass: "inetOrgPerson" # ldap 用户 objectClass 配置
    loginId: "uid" # ldap 用户惟一 id，用来作为登录的 id
    userDisplayName: "cn" # ldap 用户名，用来作为显示名
    email: "mail" # ldap 邮箱属性
  filter: # 配置过滤，目前只支持 memberOf
    memberOf: "cn=apollo,ou=groups,dc=domain,dc=com" # 只允许 memberOf 属性为 apollo 的用户访问
```

```bash
$ vi portal/scripts/startup.sh
```

```bash
# 添加一行
export JAVA_OPTS="$JAVA_OPTS -Dspring.profiles.active=github,ldap"
```

* 管理员设置，找到MySQL表ApolloPortalDB.ServerConfig，修改superAdmin，值是OpenLDAP中的用户uid(User Name)。
* 在openldap中添加组`cn=apollo,ou=groups,dc=domain,dc=com`，objectClass=groupOfNames。并添加用户。

## apollo文件夹权限

```bash
$ chmod a+x /usr/local/apollo/adminservice/scripts/*
$ chmod a+x /usr/local/apollo/configservice/scripts/*
$ chmod a+x /usr/local/apollo/portal/scripts/*
$ chmod -R o-rwx /usr/local/apollo
$ chown -R apollo:apollo /usr/local/apollo
```

## 导入MySQL

```bash
$ curl -o apollo-1.4.0.tar.gz https://github.com/ctripcorp/apollo/archive/v1.4.0.tar.gz
$ tar -zxf apollo-1.4.0.tar.gz
$ cd apollo-1.4.0
$ mysql < apollo-1.4.0/scripts/db/migration/configdb/V1.0.0__initialization.sql
$ mysql < apollo-1.4.0/scripts/db/migration/portaldb/V1.0.0__initialization.sql
```

在MySQL中增加apollo用户，并赋予`ApolloConfigDB`和`ApolloPortalDB`两个库的权限。

## 日志文件夹

```bash
$ install -d /var/log/apollo
$ chown apollo:apollo /var/log/apollo
$ chmod 0775 /var/log/apollo
```

```bash
$ vi adminservice/scripts/startup.sh
```

```conf
LOG_DIR=/var/log/apollo/adminservice
```

```bash
$ vi configservice/scripts/startup.sh
```

```conf
LOG_DIR=/var/log/apollo/configservice
```

```bash
$ vi portal/scripts/startup.sh
```

```conf
LOG_DIR=/var/log/apollo/portal
```

## systemd服务脚本

### adminservice

```bash
$ vi /usr/lib/systemd/system/apollo-adminservice.service
```

```conf
[Unit]
Description=The Apollo Admin Service
After=syslog.target network.target

[Service]
Type=simple
User=apollo
Group=apollo
PIDFile=/usr/local/apollo/adminservice/apollo-adminservice_usrlocalapolloadminservice.pid
ExecStart=/usr/local/apollo/adminservice/scripts/startup.sh
ExecStop=/usr/local/apollo/adminservice/scripts/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
$ systemctl enable apollo-adminservice
```

### configservice

```bash
$ vi /usr/lib/systemd/system/apollo-configservice.service
```

```conf
[Unit]
Description=The Apollo Config Service
After=syslog.target network.target

[Service]
Type=simple
User=apollo
Group=apollo
PIDFile=/usr/local/apollo/configservice/apollo-configservice_usrlocalapolloconfigservice.pid
ExecStart=/usr/local/apollo/configservice/scripts/startup.sh
ExecStop=/usr/local/apollo/configservice/scripts/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
$ systemctl enable apollo-configservice
```

### portal

```bash
$ vi /usr/lib/systemd/system/apollo-portal.service
```

```conf
[Unit]
Description=The Apollo Portal
After=syslog.target network.target

[Service]
Type=simple
User=apollo
Group=apollo
PIDFile=/usr/local/apollo/portal/apollo-portal_usrlocalapolloportal.pid
ExecStart=/usr/local/apollo/portal/scripts/startup.sh
ExecStop=/usr/local/apollo/portal/scripts/shutdown.sh

[Install]
WantedBy=multi-user.target
```

```bash
$ systemctl enable apollo-portal
```

## 启动服务

```bash
$ systemctl start apollo-configservice
$ systemctl start apollo-adminservice
$ systemctl start apollo-portal
```

## 访问

http://localhost:8070

