描述 mysql 中的数据类型，参考 [field_types](https://dev.mysql.com/doc/dev/mysql-server/latest/field__types_8h.html)

# 字符串类型

## varchar 和 char

- 空间

    varchar 是变长的，char 是定长的。因此，如果某个字段的大小是固定的，则应该使用 char 类型。如果字段的平均长度跟最大长度差距很大的时候，应该考虑使用 varchar 类型。

    此外，innodb compact 格式还会为 varchar 类型花费 1 - 2 字节用来存储实际长度。 具体规则为：

        - 如果类型最大长度 (字符个数 * 字符最大字节数) <= 255，则使用一个字节
        - 否则：
          - 如果实际长度 <= 127， 则使用一个字节
          - 否则，使用两个字节


    实际上，对于 char 类型，如果字符集是变长字符集，字段长度范围为: 字符个数 * [单字符最小字节数， 单字符最大字节数]。长度也是变长的，此时，innodb 底层也会存储 char 类型的长度信息。
    不过为了性能考虑，innodb 会保证 char 类型的最小长度为: 字符个数 * 单字符最小字节数。 不足的 **填充为空格，在查找的时候会自动去掉尾部空格**。

- 性能

    char 类型要比 varchar 好一些。因为，varchar 长度是不固定的，因此，如果更新的时候，新的长度大于旧的长度，可能会触发页分裂操作，影响性能并导致页碎片问题。

## varbinary 和 binary

varbinary 和 binary 存储的二进制数据。binary 与 char 不同的地方在于：

    - 长度

        binary 的长度是固定的，因此底层并不会存储字段的实际长度。这是与变长字符集的 char 不同的地方。

    - 填充

        binary 补充的是 0 字节，在查找的时候并不会去掉尾部填充字节。 char 填充的是空格，查找的时候会去掉尾部空格。

## blob 和 text

如果要存储超大长度的字符串类型，就可以考虑 blob 和 text 类型，blob 是二进制类型，text 是字符串类型。

此外，因为 blob 和 text 可能会很大，因此，在进行列排序的时候，只对每个列的最前 `max_sort_length` 字节进行排序。

## 使用 enum 代替 字符串类型

如果某个字段的值是固定的某几个字符串，则可以考虑使用 `enum` 类型代替字符串类型。

innodb 底层维护了一个 枚举字符串列表，行中存储的是对应字符串的下标。

- 这样的优势是：

    - 更小的空间

        只存储下标，不用存储字符串。

    - 更好的性能

        不过，当 enum 类型 和 字符串类型之间进行 比较的时候，性能会差一些。因为需要通过查找枚举字符串列表获取真正的字符串。不过在 enum 之间比较的时候，性能是比字符串之间比较更好。

- 缺点：

    如果字符串要发生变更，需要通过 `alter table` 的方式添加。
    
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


