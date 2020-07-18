## Ubuntu常用操作 ##

### 用户维护 ###

新增用户

```bash       
	sudo adduser <username>
```

加入sudo组

```bash
	sudo usermod -aG sudo <username>
```

修改用户的密码   

```bash
	sudo passwd <username>
```
### Host维护 ###

修改`host`名称

```bash
	sudo hostname <newhostname>
```

