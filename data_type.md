描述 mysql 中的数据类型，参考 [field_types](https://dev.mysql.com/doc/dev/mysql-server/latest/field__types_8h.html)

# datetime 与 timestamp 的区别

不管是 datetime 还是 timestamp， mysql 中时间的展示格式都是 'YYYY-MM-DD hh:mm:ss'

- 时间范围

    - datetime

        '1001-01-01 00:00:00' - '9999-12-31 23:59:59'

    - timestamp

        '1970-01-01 00:00:01' - '2038-01-19 03:14:07' UTC


- 存储格式

    - datetime

        把日期和时间封装为格式 **YYYYMMDDHHMMSS** 的整数中，使用 8 字节的存储空间。

    - timestamp

        底层存储的是 unix 时间戳，即自 '1970-01-01 00:00:00' 以来的秒数，使用 4 字节的存储空间。

- 时区

    - datetime

        与时区无关，写入什么，读取的就是什么。

    - timestamp

        与时区有关，在写入的时候，首先会根据当前的 zone 转化为 utc 再存入到 mysql 中。 读取的时候，根据当前 zone 将 utc 转化为时区对应的时间返回。

        那么如何设置时区呢？时区由系统变量 **** 指定。

- 时间精度

mysql 支持时间最大精度为 微秒。

    - 定义列

        通过语法 **type_name(fsp)** 来定义携带 fractional second 的 列。 比如：TIMESTAMP(3) 表示精度为毫秒。默认值为 0，但是 sql 标准是 6， mysql 这么做是为了跟之前的版本兼容。

    - 四舍五入

        如果插入值的精度被列的精度高，则四舍五入。 eg: 在 TIMESTAMP(3) 列插入 '2000-01-01 12:00:00.8888'，实际存入的结果为 '2000-01-01 12:00:00.889'

    - 产生时间函数

        比如 NOW()，默认产生的时间没有 fractional part，但是可以通过参数 NOW(0-6) 来指定产生的时间的精度。


# 时区如何设置

详情参考[time zone](https://dev.mysql.com/doc/refman/8.0/en/time-zone-support.html)

- 系统时区

    服务器在启动的时候，会判断当前主机所在的时区，并使用该值初始化 **system_time_zone** 系统变量。

    可以通过以下两种方式来显示的指定主机所在的时区。

    - TZ

        在通过 mysqld 启动服务器的时候，可以通过环境变量 **TZ** 来设置系统时区。

    - --timezone

        在通过 mysqld_safe 启动服务器的时候，可以通过命令行选项 **--timezone** 设置系统时区。

- 服务器时区

    全局变量 time_zone 表示服务器当前的时区，默认值为 **SYSTEM** 表示跟 **system_time_zone** 一致。可以通过以下三种方式来更改 global time_zone 的值。

    - 命令行

        --default-time-zone='timezone'

    - 配置文件

        default-time-zone='timezone'

    - 运行时

        SET GLOBAL time_zone = timezone;

- 会话时区

    会话维度也有自己的时区，默认值跟 global time zone 一致，不过可以在运行时修改：

    SET time_zone = timezone;


