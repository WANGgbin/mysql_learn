描述 mysql 中权限相关内容。

# 用户

- 定义用户

    `create user 'user_name'@'host_name' identified with 'authentication_method' by 'password';`

    authentication_method 是用户的密码验证插件，默认值是 `caching_sha2_password`。可以在命令行选项或者配置文件中通过 `default-authentication-plugin` 设置默认值。
    
    特别注意：**authentication_method 是用户维度的，用户在登录的时候，服务器通过该方法来验证用户身份**。
    
    通过查看 mysql.user 的 `plugin` 列查看某个用户对应的密码验证插件。

- 修改用户

    - 修改密码

        `alter user 'user_name'@'host_name' identified by 'new_password';`

    - 修改用户名

        `rename user 'user_name'@'host_name' to 'new_name'@'host_name';`

- 删除用户

    `drop user 'user_name'@'host_name';`


- 给用户授权

    grant 相关命令可以参考: [grant](https://www.runoob.com/note/19873)

    - 库权限

        `grant all on 'db_name.*' to 'user_name'@'host_name';`

        授权以后，用户就有 db_name 的所有权限了。判断用户是否有某个库/表的权限，可以通过 show databases; show tables; 来查看。

    - 表权限

        `grant all on 'db_name.table_name' to 'user_name'@'host_name';`

    - 删除权限

        `revoke all on 'db_name[.table_name]' from 'user_name'@'host_name';`

    - 查看权限

        `show grants for 'user_name'@'host_name';`