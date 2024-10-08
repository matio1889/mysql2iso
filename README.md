# mysql2iso

该工具可用于在线迁移库、表数据，也可用于数据库之间同步数据，比如冷数据同步、数据合并、跨机房同步等，可结合mysqldump或其他备份工具导出利用binlog同步的方式进行追加数据，也可以直接对表数据进行在线迁移，该工具基于python3进行开发，可以用于学习及工作上的需求。



# 功能

 1. 通过binlog追加dml数据目标库 
 2. 可过滤性，包括库、表、线程ID、操作类型(不同表可以有不同的过滤类型)  
 3. 记录同步状态，支持断点续传 
 4. 支持在线导出表，可多线程/单线程 
 5. 支持基于zookeeper服务注册的集群模式
 6. 同步binlog支持基于库、表为单位并发入库
 7. 数据同步支持回环控制  
 8. 支持mysql之间的同构/异构，以及mysql到phonix/postgresql的同构/异构数据同步(合并字段的异构只支持导出数据，不支持binlog同步)
 9. 支持单独定义某个表的过滤属性


## 配置项介绍：

### iso.conf:  
	include: 任务配置文件所在目录，务修改 
	config： 单机模式下使用的任务配置文件 
	cluster： 集群模式开关，可配置项True/False ，首字母需大写 
	host： 配置本机IP地址，基于leader模式选举需要使用的IP地址，这里手动配置是为了防止一个服务器有多个IP的情况 (功能未完成可不管)
	cluster_type： 集群模式类型，可配置zk_mode/leader_mode，相对应的就是两种不同实现类型 	
	zk_hosts： zookeeper地址，多个使用逗号分隔，如10.1.1.1:2181,10.1.1.2:2181....，仅当使用zk_mode时才生效 
	cluster_nodes： 基于leader_mode集群模式时配置结群中节点信息，多个依然用逗号分隔任务配置文件(功能未完成，可不管)  
### include(目录配置文件):
	global: 
		 server_id: 注册为slave时使用的server_id，多个同步任务master为同一实例时不能相同，多个任务使用同步状态 库时也不能相同，用于断点续传及任务接管时获取数据的关键字段 
		 full： 是否在线导出库、表，配置项True/False，首字母需大写，默认为False,如果开启该参数将首先导出数据这将 导致后面设置的daemon、binlog相关项失效，在确认是在线导出表时才开启该参数 	
		 threads： 在线导出线程数，默认单线程 
		 ignore： 过滤binlog中的操作类型不做同步，可以选delete,insert,update 
		 ignore_thread： 过滤某个thread_id产生的binlog不做同步 
		 ssl： 启用ssl加密链接，跨机房走公网同步时强烈建议使用ssl 
		 cert: ssl加密验证文件，使用绝对路径 （可放conf/keys下）
		 key： ssl加密验证文件，使用绝对路径 （可放conf/keys下） 
		 daemon： 表示直接从dump_status中获取记录的gtid/binlog进行同步操作,默认False，可配置True/False 
		 
	source: 
		host: 源库IP地址 
		port: 数据库端口 
		socket： 连接mysql时使用的socket文件，需绝对路径，当host为localhost时可配置 
		user_name/user_password： 用户名密码 
		databases： 同步的库名配置，多个以逗号分隔，该项必须配置 
		tables： 该配置可选，同步的表名配置，多个以逗号分开，如果不配置该项就同步库下面的所有表，如果配置了该选项只 会同步这些表，不会管是在配置的那个库下面，该选项仅适合于单库下表的过滤 
		binlog_file： slave注册时同步开始的binlog文件名 
		start_position： slave注册时同步开始的position 
		auto_position: 同mysql同步配置，可选True/False两项，为True表示使用GTID，默认为False，使用binlog+pos 的模式 
		gtid: gtid模式需要设置的gtid值 
		
	destination： 
		dthreads: 并发入库线程数，默认为单线程 
		dhost： 目标库IP地址 ‘
		dport： 目标库端口 
		duser/dpassword： 目标库用户名密码 
		binlog： 是否记录binlog,可配置True/False，默认为False，在链接创建时将设置sql_log_bin=0 
		jar: 目标库为phoenix时指定链接用的jar包 (不用配置)
		jar_conf： 链接phoenix时需配置的参数 （不用配置）
		
	status：同步状态保存配置项（mysql） 
		shost： 状态库IP地址 
		sport： 状态库端口 
		suser/spassword： 状态库账号密码，需要创建库、表权限 
		binlog: 是否记录binlog,可配置True/False，默认为False，在链接创建时将设置sql_log_bin=0 
		
	attach: 单独过滤操作 
		ignores： 过滤操作配置,以列表的方式配置，多个以逗号分割
		
	map:  
		source_db: 配置库名对应异构情况，见map_example

## 实现原理：
### 在线导出：
	在线导出采用了分段的方式，多线程分配不同数据段，为确保多个线程数据不重复，所以需要表有索引，
	选择索引的优先级（自增->主键->唯一索引->索引），如果表没有任何索引且为多线程导出，程序会
	直接报错并退出
### 数据同步：
	实现mysql replication slave端获取binlog数据，通过解析其中的row数据进行解析并放入queue，
	所以mysqlbinlog数据必须为row模式，消费线程从队列获取sql并按100个GTID事务或10秒位单分块，
	再按库、表分组块内数据放入入库队列，入库线程获取数据重现操作到目标库，块内数据分组后的数据按
	GTID事务顺序执行，且一次性提交提高性能，为控制单表数据入库顺序，在获取数据准备入时会首先获取
	一个内部锁防止队列前后相同表数据被不同线程并发执行
### 过滤：
	每个操作在row_event之前都会有table_map_event、query_event，其中包含该操作的库、表、线
	程ID等信息，过滤及获取元数据都在该event下完成
### 回环控制：
	追加数据时会首先写入当前事务binlog位置，这样使得同步程序产生的binlog第一个操作就是标签操作，
	这样再使用同步程序级联操作时就可以通过table_map_event获取到库名、表名判断，如果是标签库操作
	将直接跳过该GTID事务，以达到回环控制的目的，同时该标签数据也是宕机重启binlog进行判断的标准，
	精确到库、表位单位判断执行到的位置，如果想手动操作，只需一个事务中首先操作标签库的的任意表可达
	到该效果

## 注意事项：
	

 1. 状态数据保存到一个mysql数据库中，账号需要create权限，状态库名为dump2db
 2. 如果目标库为mysql,目标库账号需要information_schema.columns表的查询权限以及对应库表的操作权限
 3. 源库replication slave权限，如果在线导出需要对应库表的查询权限
 4. 只实现了追加binlog中row数据操作，binlog必须为row格式
 5. 如果多个任务或多线程执行同一任务时使用同样的dump2db需要使用不同的serverid,因为dump2db以serverid进行区分
 6. 在线导出数据会短暂全库读锁，在初始化完成所有链接并获取到当前binlog信息时会释放
 7. 全量导出途中如果某一个线程在操作途中发生错误将退出整个任务，以免发生数据不一致的情况
 8. 导出数据到目标库会首先删除对应的目标表，并创建一个空表
 9. 多线程导出选择索引的优先级（自增->主键->唯一索引->索引），如无索引的表将直接退出
 10. 数据提交及状态提交使用的不同链接、不同库无法保证在提交的间隙不会宕机，所以采用了提交数据再提交状态，首先保证数据入库，同时在入库时写入当前执行的binlog位置，如果发生宕机重启或接管任务，会通过该数据进行对比跳过重复binglog数据 
 11. 跨区域同步需要先在目标库执行create目录下文件创建回环控制的标签表
 12. binlog同步状态需等分块数据全入库提交才会保存，所以该dump_status保存的binlog信息落后于真正入库的数据
 13. binlog解析队列最大长度位2048个GTID事务，如果master执行了大事务就会阻塞该表后面的事务执行，如果时间超过60秒将导致队列超限直接退出程序
 14. 如果开启异构同步，最终入库关系由异构配置文件配置的内容控制，如果在配置文件中同步db1，但在异构配置中未配置db1的相关项将不会对db1的数据进行操作

## 需求的包：

pymysql、struct、IPy、kazoo、numpy、jaydebeapi、psycopg2、phoenixdb

