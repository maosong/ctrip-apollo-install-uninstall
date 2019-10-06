# 卸载

## 关闭服务

```bash
$ systemctl stop apollo-configservice
$ systemctl stop apollo-adminservice
$ systemctl stop apollo-portal
```

## 删除日志文件夹

```bash
$ rm -rf /var/log/apollo
```

## 删除apollo文件夹

```bash
$ rm -rf /usr/local/apollo
```

## 删除apollo用户

```bash
$ userdel -rf apollo
```

