Jdbc input plugin
描述
此插件能创建任何JDBC接口的数据提取到logstash中, 您可以使用cron语法, 定期计划摄取, 或者运行查询一次以将数据加载到Logstash中, 结果集中数据每一行成为一个事件, 结果集中的列将转换为事件中的字段.

三个工作阶段
image

input 数据输入端, 可以接收来自任何地方的源数据。

file : 从文件中读取
syslog : 监听在514端口的系统日志信息, 并解析成RFC3164格式.
redis : 从redis-server list 中获取.
beat : 接收来自Filebeat的事件
filter 数据中转层，主要进行格式处理，数据类型转换、数据过滤、字段添加，修改等，常用的过滤器如下。

grok : 通过正则解析和结构化任何文本. Grok 目前是logstash最好的方式对非结构化日志数据解析成结构化和可查询化.logstash内置了120个匹配模式，满足大部分需求.
mutate: 在事件字段执行一般的转换. 可以重命名、删除、替换和修改事件字段.
drop : 完全丢弃事件, 如debug事件.
clone : 复制事件, 可能添加或删除字段.
geoip : 添加有关ip地址地理位置信息.
output 是logstash工作的最后一个阶段，负责将数据输出到指定位置，兼容大多数应用，常用的有:

elasticsearch: 发送事件数据到 Elasticsearch, 便于查询, 分析, 绘图.
file: 将事件数据写入到磁盘文件上.
mongodb : 将事件数据发送至高性能NoSQL mongodb, 便于永久存储, 查询, 分析, 大数据分片.
redis : 将数据发送至 redis-server, 常用于中间层暂时缓存
graphite : 发送事件数据到 graphite.
statsd : 发送事件数据到 statsd.
基本配置
logstash使用{ }来定义配置区域，区域内又可以包含其插件的区域配置.

# 最基本的配置文件定义，必须包含input 和 output.
input{
stdin{ }
}

output{
stdout{
codec=>rubydebug
}
}

# 如果需要对数据进操作，则需要加上filter段
input{
stdin{ }
}

filter{
# 里面可以包含各种数据处理的插件，如文本格式处理 grok、键值定义 kv、字段添加、
# geoip 获取地理位置信息等等...

}

output{
stdout{
codec=>rubydebug
}
}


# 可以定义多个输入源与多个输出位置
input{
stdin{ }

    file{
        path => ["/var/log/message"]
        type => "system"
        start_position => "beginning"
    }
}

output{
stdout{
codec=>rubydebug
}

    file {
        path => "/var/datalog/mysystem.log.gz"
        gzip => true
    }
}
调度
此插件可以根据指定的时期运行, 由rufus-scheduler提供语法支持. 语法类似cron, 具有特定于Rufus的一些扩展.(如: 时区等.)
例子:

语法	效果
* 5 * 1-3 *	将于1月至3月的每一天凌晨5点执行
  0 * * * *	将在每小时的第0分钟执行
  0 6 * * * America/Chicago	将每天上午6:00(UTC/GMT-5)执行
  状态
  该插件将以sql_last_value存储在已配置的元数据文件中的形式保留参数 last_run_metadata_path. 执行查询后, 将使用当前值更新此文件 sql_last_value. 下次管道启动时, 将通过读取此文件更新此值, 如果将clean_run 设置为true, 则将忽略此值, 并将其sql_last_value 设置为时间戳起始 1970-1-1, 如果use_column_value 为true, 则为0, 就好像没有查询过一样.

处理大型结果集
许多Jdbc驱动程序使用该fetch_size参数来限制从游标到客户端缓存中一次预取的结果数, 然后从结果集中检索更多结果. 这是使用jdbc_fetch_size配置选项在此插件中配置的. 此插件默认情况下未设置提取大小, 因此将使用特定驱动程序的默认大小.

使用
以下为使用Mysql数据库获取数据的示例, 首先, 我们将适当的Jdbc驱动程序放在指定路径中, 使用连接连接数据库, 并在指定的调度中执行sql语句.

// 数据输入项
input {
// 数据输入接口类型
jdbc {
// jar包驱动程序所在路径
jdbc_driver_library => "/usr/share/logstash/lib/mysql-connector-java-5.1.47.jar"
// 驱动类地址
jdbc_driver_class => "com.mysql.jdbc.Driver"
// 驱动连接信息
jdbc_connection_string => "jdbc:mysql://10.101.22.99:3306/TestData?useSSL=false"
// 数据库用户名
jdbc_user => "root"
// 数据库密码
jdbc_password => "root"
// Jdbc启用分页, 这导致sql语句被分解为多个查询.
// 每个查询将使用限制和偏移量来共同检索完整的结果集.
// 限制大小设置为 jdbc_page_size.
jdbc_paging_enabled => "true"
// 是否应保留先前的运行状态
clean_run => false
// JDBC页面大小 默认值是 100000
jdbc_page_size => "50000"
// 上次运行时文件的路径 默认值是 "$HOME/.logstash_jdbc_last_run"
last_run_metadata_path => "/usr/share/logstash/pipeline/last_id"
// 是否保存状态到 last_run_metadata_path 所在文件
record_last_run => true
// 设置为ture时, 使用定义的tracking_column值作为 :sql_last_value.
// 设置为false时, :sql_last_value反映上次查询的时间
use_column_value => false
// 当use_column_value 设置为true的时候, 需要跟踪的值列, 一般为自增主键.
tracking_column => "id"
// 包含需要执行语句的路径.
statement_filepath => "/usr/share/logstash/pipeline/All.sql"
}
}
// 数据输出项
output {
// 输出接口类型
elasticsearch {
// 连接地址
hosts => "10.101.22.99:9200"
// ES 索引
index => "news"
// ES 类型
document_type => "message"
// ES 文档id设置对象字段
document_id => "%{id}"
// ES 连接用户名
user => "elastic"
// Es 连接密码
password => "changeme"
}
stdout {
// 输出在屏幕时, 编码器格式
codec => json_lines
}
}
配置SQL语句
此输入需要sql语句, 这可以通过字符串形式的语句选项传入, 也可以从文件(statement_filepath)中读取, 当SQL语句很大或很难在配置中提供时, 通常使用文件选项, file选项仅支持一个SQL语句. 该插件只接收其中一个选项. 它无法从文件以及statement配置参数中读取语句.

配置多个SQL语句
当需要查询和从不同数据库表或视图中提取数据时, 配置多个SQL语句非常有用.可以为每个语句定义单独的logstash配置文件, 也可以在单个配置文件中定义多个语句, 在单个Logstash配置文件中使用多个语句时, 必须将每个语句定义为单独的jdbc输出(包括jdbc驱动程序, 连接字符串和其他必须参数).
请注意, 如果任何语句使用该sql_last_value参数 (例如, 仅摄取自上次运行以来更改的数据), 则每个输入应定义其自己的last_run_metadata_path参数. 如果不这样做将导致不期望的行为, 因为所有输入都将其存储到相同(默认)元数据文件中, 从有效地覆盖彼此sql_last_value.

预定义参数
某些参数是内置的, 可以在查询中使用, 清单:
sql_last_value 用于计算要查询的行的值. 在运行任何查询之前, 将其设置为1970-1-1, 如果use_column_value设置为true, 且tracking_column设置为0, 则为0. 在后续查询运行后, 他会相应更新.

Jdbc输入配置选项
clean_run

值类型是 boolean
默认值是 false
是否保留先前的运行状态

columns_charset

值得类型是 hash
默认值是 {}
特定列的字符编码, 此项将覆盖: charset指定列的选项.

columns_charset => {"column0" => "ISO-8859-1}
这只会将ISO-8859-1的column0转换为原始编码.

connection_retry_attempts

值类型是 number
默认值是 1
尝试连接数据库的最大次数

connection_retry_attempts_wait_time

值类型是 number
默认值是 0.5
连接尝试之间休眠的秒数

jdbc_connection_string

这是必需的设置
值类型是 string
此设置没有默认值
Jdbc连接配置字符串

jdbc_default_timezone

值类型是 string
此设置没有默认值
时区转换, SQL不允许时间戳字段中的时区数据. 此插件将自动将SQL时间戳字段转换为Logstash时间戳, 采用ISO8601格式的相对UTC时间.
使用此设置将手动分配指定的时区偏移量, 而不是使用本地计算机的时区设置, 但必需使用规范时区 America/Denver

jdbc_driver_class

这是必需的设置
值类型是 string
此设置没有默认值
如果使用的是Mysql Jdbc, 则需要加载Jdbc驱动程序类,如: "com.mysql.jdbc.Driver"

jbdc_driver_library

值类型是 string
此设置没有默认值
试图将Jdbc逻辑抽象为mixin, 以便在其他插件中进行重用(input/output) 当有人包含此模块时, 调用这些方法添加给定的基础. Jdbc驱动程序到第三方驱动库的路径. 如果需要多个库, 可以用都好分割

jdbc_fetch_size

值类型是 number
此设置没有默认值
Jdbc获取大小, 如果没有提供, 将使用相应的驱动程序默认值.

jdbc_page_size

值类型是 number
默认值是 100000
Jdbc 页面大小

jdbc_paging_enabled

值类型是boolean
默认值是false
Jdbc启用分页 这将导致sql语句被分解为多个查询, 每个查询将使用限制和偏移来共同检索完整的结果集. 限制大小设置为jdbc_page_size.

jdbc_password

值类型是 password
此设置没有默认值
Jdbc连接密码

jdbc_password_filepath

值类型是 filePath(系统路径地址)
此设置没有默认值
Jdbc密码文件名

jdbc_pool_timeout

值类型是 number
默认值是 5
连接池配置, 在引发PoolTimeoutError之前等待获取连接的秒数

jdbc_user

这是必须的设置
值类型是 string
此设置没有默认值
Jdbc连接用户

jdbc_validate_connnection

值类型是 boolean
默认值是 false
连接池的配置, 使用前验证连接

jdbc_validation_timeout

值类型是 number
默认值是 3600
连接池配置, 验证连接的频率 (以秒为单位)

last_run_metadata_path

值类型是 string
默认值是 $HOME/.logstash_jdbc_last_run"
上次运行时文件路径

lowercase_column_names

值类型是 boolean
默认值是 true
是否强标识字段的小写

parameters

值类型是 hash
默认值是 {}
例如, 查询参数的哈希值 {"target_id" => "321"}

record_last_run

值类型是 boolean
默认值是 true
是否保存状态 到 last_run_metadata_path 所配置的路径

schedule

值类型是 string
此设置没有默认值
以Cron格式定期运行语句的时间表, 如 : "* * * * *" (每分钟执行的查询, 分钟) 默认没有执行计划, 那么执行sql语句只执行一次.

sequel_opts

值类型是 hash
默认值是 {}
General/Vendor-specific 特定的续集配置选项
可选连接池配置示例 max_connections- 连接池的最大连接数
特定选项的示例https://github.com/jeremyevans/sequel/blob/master/doc/opening_databases.rdoc

sql_log_level

值可以是任何的 : fatal, error, warn, info, debug
默认值是 "Info"
记录SQL 查询的日志级别, 接受的值是常见的致命, 错误, 警告, 信息, 和调试. 默认值为 info.

statement

值类型是 string
此设置没有默认值
指定需要执行的基本语句.

statement_filepath

值类型是 filePath(系统路径地址)
此设置没有默认值
包含要执行的语句的文件路径.

tracking_column

值类型是 string
此设置没有默认值
如果use_column_value 设置为, 则要跟踪其值的列true

tracking_column_type

值可以是任何的 : numeric, timestamp
默认值是 "numeric"
跟踪列的类型. 目前只有"数字"和 "时间戳"

use_column_value

值类型是 boolean
默认值是 false
设置为时true, 使用定义的tracking_column值作为 :sql_last_value, 设置为时false, :sql_last_value 反映上次执行查询的时间.

常用选项
所有输入(input)插件都支持以下选项

add_field

值类型是hash
默认值是 {}
向事件添加字段

codec

值类型是 codec
默认值是"plain
用于输入数据的编解码器, 输入编解码是一种在数据进入输入之前解码数据的便捷方法, 无需在Logstash管道中使用单独的过滤器.

enable_metric

值类型是 boolean
默认值是 true
默认情况下, 禁用或启用次特定插件实例的度量标准记录我们会记录所有可用的度量标准, 但您可以禁用特定插件的度量标准收集.

id

值得类型是 string
此设置没有默认值
ID为插件配置添加唯一. 如果未指定ID, Logstash 将生成一个ID, 强烈建议配置中设置此ID. 当您有两个或更多相同类型的插件时, 这尤其有用, 例如, 如果你有两个jdbc输入. 这种情况下, 添加命名ID 将有助于在使用监视API时监视Logstash

jdbc { id => "my_plugin_id"}
tags

值类型是 array
此设置没有默认值
为您的活动添加任意数量的任意标签
这有助与以后处理

Notice : logstash jdbc-plugin 时间缺漏
问题可能在于order by last_update_time 获取最后一条的时间.