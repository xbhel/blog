## 创建用户

使用 `useradd` 来创建一个新用户，默认会在 */home* 下创建一个同名目录作为用户主目录，同时也会为该用户创建一个同名的组。通过 `man useradd` 来查看帮助信息。

```bash
# 创建一个名为 xbhel 的用户
$ useradd xbhel

# 查看用户所在组
$ groups xbhel
#> xbhel

# 给用户追加一个组 root
$ usermod -a -G root xbhel

# 再次查看用户所在组
$ groups xbhel
#> xbhel root

# 重新指定用户组，可利用此删除多余组
usermod -G xbhel xbhel

# 再次查看用户所在组
$ groups xbhel
#> xbhel

# 设置密码
$ passwd xbhel
```

## 添加 sudo 权限

使用 `visudo` 命令编辑 */etc/sudoers* 文件来为用户添加 `sudo` 权限，在最后一行添加如下内容：

```bash
xbhel	ALL=(ALL)	ALL
```

