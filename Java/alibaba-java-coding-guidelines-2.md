# 阿里巴巴Java开发手册——异常处理、MySQL 数据库


## 二、异常日志

### (一) 异常处理

- Java类库中定义的可以通过预检查方式规避的`RuntimeException`异常不应该通过`catch`的方式来处理，比如：`NullPointerException`，`IndexOutOfBoundsException`等等【说明：无法通过预检查的异常除外，比如，在解析字符串形式的数字时，可能存在数字格式错误，不得不通过`catch NumberFormatException`来实现】
> 正例：`if (obj != null) {...}`
反例：try { obj.method(); } catch (NullPointerException e) {…}
- 异常不要用来做流程控制，条件控制【说明：异常设计的初衷是解决程序运行中的各种意外情况，且异常的处理效率比条件判断方式要低很多】
- `catch`时请分清稳定代码和非稳定代码，稳定代码指的是无论如何不会出错的代码。对于非稳定代码的`catch`尽可能进行区分异常类型，再做对应的异常处理【说明：对大段代码进行`try-catch`，使程序无法根据不同的异常做出正确的应激反应，也不利于定位问题，这是一种不负责任的表现。】
- 捕获异常是为了处理它，不要捕获了却什么都不处理而抛弃之，如果不想处理它，请将该异常抛给它的调用者。最外层的业务使用者，必须处理异常，将其转化为用户可以理解的内容
- 有`try`块放到了事务代码中，`catch`异常后，如果需要回滚事务，一定要注意手动回滚事务
- `finally`块必须对资源对象、流对象进行关闭，有异常也要做`try-catch`。【如果JDK7及以上，可以使用`try-with-resources`方式】
- 不要在`finally`块中使用`return`【说明：`try`块中的`return`语句执行成功后，并不马上返回，而是继续执行`finally`块中的语句，如果此处存在`return`语句，则在此直接返回，无情丢弃掉`try`块中的返回点。】
```java
// 反例
private int x = 0;
public int checkReturn() {
  try {
    // x 等于 1，此处不返回
    return ++x;
  } finally {
    // 返回的结果是 2
    return ++x;
  }
}
```
- 捕获异常与抛异常，必须是完全匹配，或者捕获异常是抛异常的父类。【说明：如果预期对方抛的是绣球，实际接到的是铅球，就会产生意外情况。】
- 在调用`RPC、二方包、或动态生成类`的相关方法时，捕捉异常必须使用`Throwable`类来进行拦截。【说明：通过反射机制来调用方法，如果找不到方法，抛出`NoSuchMethodException`。什么情况会抛出`NoSuchMethodError`呢？二方包在类冲突时，仲裁机制可能导致引入非预期的版本使类的方法签名不匹配，或者在字节码修改框架（比如：ASM）动态创建或修改类时，修改了相应的方法签名。这些情况，即使代码编译期是正确的，但在代码运行期时，会抛出`NoSuchMethodError`。】
- 方法的返回值可以为`null`，不强制返回空集合，或者空对象等，必须添加注释充分说明什么情况下会返回`null`值【说明：本手册明确防止`NPE`是`调用者的责任`。即使被调用方法返回空集合或者空对象，对调用者来说，也并非高枕无忧，必须考虑到远程调用失败、序列化失败、运行时异常等场景返回`null`的情况。】
- 防止`NPE`，是程序员的基本修养，注意`NPE`产生的场景
> 1）返回类型为基本数据类型，`return`包装数据类型的对象时，自动拆箱有可能产生 NPE。【反例：`public int f() { return Integer 对象}`，如果为`null`，自动解箱抛`NPE`】  
2）数据库的查询结果可能为`null`  
3）集合里的元素即使`isNotEmpty`，取出的数据元素也可能为`null`。  
4）远程调用返回对象时，一律要求进行空指针判断，防止`NPE`  
5）对于`Session`中获取的数据，建议进行`NPE`检查，避免空指针。  
6）级联调用`obj.getA().getB().getC()；`一连串调用，易产生`NPE`。正例：使用JDK8的`Optional`类来防止NPE问题。
- 定义时区分`unchecked / checked`异常，避免直接抛出`new RuntimeException()`，更不允许抛出`Exception`或者`Throwable`，应使用有业务含义的自定义异常。推荐业界已定义过的自定义异常，如：`DAOException / ServiceException`等。
- 对于公司外的`http/api`开放接口必须使用“错误码”；而应用内部推荐异常抛出；跨应用间`RPC`调用优先考虑使用`Result`方式，封装`isSuccess()`方法、“错误码”、“错误简短信息”【说明：关于`RPC`方法返回方式使用`Result`方式的理由：1）使用抛异常返回方式，调用方如果没有捕获到就会产生运行时错误。2）如果不加栈信息，只是new自定义异常，加入自己的理解的`error message`，对于调用端解决问题的帮助不会太多。如果加了栈信息，在频繁调用出错的情况下，数据序列化和传输的性能损耗也是问题。】
- 避免出现重复的代码（`Don't Repeat Yourself`），即DRY原则【说明：随意复制和粘贴代码，必然会导致代码的重复，在以后需要修改时，需要修改所有的副本，容易遗漏。必要时抽取共性方法，或者抽象公共类，甚至是组件化】

### (二) 日志规约
- 应用中不可直接使用日志系统（`Log4j、Logback`）中的API，而应依赖使用日志框架`SLF4J`中的API，使用门面模式的日志框架，有利于维护和各个类的日志处理方式统一
```java
import org.slf4j.Logger; 
import org.slf4j.LoggerFactory;
private static final Logger logger = LoggerFactory.getLogger(Test.class);
```
- 所有日志文件至少保存15天，因为有些异常具备以“周”为频次发生的特点。网络运行状态、安全相关信息、系统监测、管理后台操作、用户敏感操作需要留存相关的网络日志不少于6个月
- 应用中的扩展日志（如打点、临时监控、访问日志等）命名方式：`appName_logType_logName.log`。logType:日志类型，如`stats/monitor/access`等；logName:日志描述。这种命名的好处：通过文件名就可知道日志文件属于什么应用，什么类型，什么目的，也有利于归类查找【说明：推荐对日志进行分类，如将错误日志和业务日志分开存放，便于开发人员查看，也便于通过日志对系统进行及时监控。正例：`force-web`应用中单独监控时区转换异常，如：`force_web_timeZoneConvert.log`】
- 在日志输出时，字符串变量之间的拼接使用占位符的方式【说明：因为`String`字符串的拼接会使用`StringBuilder`的`append()`方式，有一定的性能损耗。使用占位符仅是替换动作，可以有效提升性能。正例：`logger.debug("Processing trade with id: {} and symbol: {}", id, symbol);`】
- 对于`trace/debug/info`级别的日志输出，必须进行日志级别的开关判断【说明：虽然在debug(参数)的方法体内第一行代码`isDisabled(Level.DEBUG_INT)`为真时（Slf4j的常见实现`Log4j`和`Logback`），就直接`return`，但是参数可能会进行字符串拼接运算。此外，如果`debug(getName())`这种参数内有`getName()`方法调用，无谓浪费方法调用的开销。】
```java
// 正例
// 如果判断为真，那么可以输出trace和debug级别的日志
if (logger.isDebugEnabled()) {
  logger.debug("Current ID is: {} and name is: {}", id, getName());
}
```
- 避免重复打印日志，浪费磁盘空间，务必在`log4j.xml`中设置`additivity=false`【正例：`<logger name="com.taobao.dubbo.config" additivity="false">`】
- 异常信息应该包括两类信息：`案发现场信息`和`异常堆栈信息`。如果不处理，那么通过关键字`throws`往上抛出【正例：`logger.error(各类参数或者对象 toString() + "_" + e.getMessage(), e);`】
- 谨慎地记录日志。生产环境禁止输出`debug`日志；有选择地输出`info`日志；如果使用`warn`来记录刚上线时的业务行为信息，一定要注意日志输出量的问题，避免把服务器磁盘撑爆，并记得及时删除这些观察日志【说明：大量地输出无效日志，不利于系统性能提升，也不利于快速定位错误点。记录日志时请思考：`这些日志真的有人看吗？看到这条日志你能做什么？能不能给问题排查带来好处？`】
- 可以使用`warn`日志级别来记录用户输入参数错误的情况，避免用户投诉时，无所适从。如非必要，请不要在此场景打出`error`级别，避免频繁报警【说明：注意日志输出的级别，`error`级别只记录`系统逻辑出错、异常或者重要的错误`信息】
- 尽量用英文来描述日志错误信息，如果日志中的错误信息用英文描述不清楚的话使用中文描述即可，否则容易产生歧义【国际化团队或海外部署的服务器由于字符集问题，使用全英文来注释和描述日志错误信息。】

## 三、单元测试

- 好的单元测试必须遵守`AIR`原则【说明：单元测试在线上运行时，感觉像空气（AIR）一样并不存在，但在测试质量的保障上，却是非常关键的。好的单元测试宏观上来说，具有自动化、独立性、可重复执行的特点。A：`Automatic`（自动化）I：`Independent`（独立性）R：`Repeatable`（可重复）】
- 单元测试应该是全自动执行的，并且非交互式的。测试用例通常是被定期执行的，执行过程必须完全自动化才有意义。输出结果需要人工检查的测试不是一个好的单元测试。单元测试中不准使用`System.out`来进行人肉验证，必须使用`assert`来验证
- 保持单元测试的独立性。为了保证单元测试稳定可靠且便于维护，单元测试用例之间决不能互相调用，也不能依赖执行的先后次序【反例：`method2`需要依赖`method1`的执行，将执行结果作为`method2`的输入】
- 单元测试是可以重复执行的，不能受到外界环境的影响。【说明：单元测试通常会被放到持续集成中，每次有代码`check in`时单元测试都会被执行。如果单测对外部环境（网络、服务、中间件等）有依赖，容易导致持续集成机制的不可用。正例：为了不受外界环境影响，要求设计代码时就把SUT的依赖改成注入，在测试时用`spring`这样的`DI`框架注入一个本地（内存）实现或者`Mock`实现】
- 对于单元测试，要保证测试粒度足够小，有助于精确定位问题。单测粒度至多是类级别，一般是方法级别【说明：只有测试粒度小才能在出错时尽快定位到出错位置。单测不负责检查跨类或者跨系统的交互逻辑，那是集成测试的领域】
- 核心业务、核心应用、核心模块的增量代码确保单元测试通过【说明：新增代码及时补充单元测试，如果新增代码影响了原有单元测试，请及时修正】
- 单元测试代码必须写在如下工程目录：`src/test/java`，不允许写在业务代码目录下【说明：源码编译时会跳过此目录，而单元测试框架默认是扫描此目录】
- 单元测试的基本目标：语句覆盖率达到70%；核心模块的语句覆盖率和分支覆盖率都要达到100%【说明：在工程规约的应用分层中提到的`DAO`层，`Manager`层，可重用度高的`Service`，都应该进行单元测试。】
- 编写单元测试代码遵守`BCDE`原则，以保证被测试模块的交付质量【B：`Border`，边界值测试，包括循环边界、特殊取值、特殊时间点、数据顺序等; C：`Correct`，正确的输入，并得到预期的结果; D：`Design`，与设计文档相结合，来编写单元测试; E：`Error`，强制错误信息输入（如：非法数据、异常流程、业务允许外等），并得到预期的结果】
- 对于数据库相关的查询，更新，删除等操作，不能假设数据库里的数据是存在的，或者直接操作数据库把数据插入进去，请使用程序插入或者导入数据的方式来准备数据【反例：删除某一行数据的单元测试，在数据库中，先直接手动增加一行作为删除目标，但是这一行新增数据并不符合业务插入规则，导致测试结果异常】
- 和数据库相关的单元测试，可以设定自动回滚机制，不给数据库造成脏数据。或者对单元测试产生的数据有明确的前后缀标识
- 对于不可测的代码在适当的时机做必要的重构，使代码变得可测，避免为了达到测试要求而书写不规范测试代码
- 在设计评审阶段，开发人员需要和测试人员一起确定单元测试范围，单元测试最好覆盖所有测试用例
- 单元测试作为一种质量保障手段，在项目提测前完成单元测试，不建议项目发布后补充单元测试用例
- 为了更方便地进行单元测试，业务代码应避免以下情况【构造方法中做的事情过多；存在过多的全局变量和静态方法；存在过多的外部依赖；存在过多的条件语句；说明：多层条件语句建议使用卫语句、策略模式、状态模式等方式重构】
- 不要对单元测试存在如下误解【那是测试同学干的事情。1.本文是开发手册，凡是本文内容都是与开发同学强相关的；2.单元测试代码是多余的。系统的整体功能与各单元部件的测试正常与否是强相关的；3.单元测试代码不需要维护。一年半载后，那么单元测试几乎处于废弃状态；4.单元测试与线上故障没有辩证关系。好的单元测试能够最大限度地规避线上故障】

## 四、安全规约

- 隶属于用户个人的页面或者功能必须进行`权限控制校验`【说明：防止没有做水平权限校验就可随意访问、修改、删除别人的数据，比如查看他人的私信内容、修改他人的订单】
- 用户敏感数据禁止直接展示，必须对展示数据进行`脱敏`【说明：中国大陆个人手机号码显示为:137****0969，隐藏中间4位，防止隐私泄露】
- 用户输入的SQL、参数严格使用参数绑定或者 METADATA 字段值限定，防止`SQL 注`入，禁止字符串拼接SQL访问数据库
- 用户请求传入的任何参数必须做有效性验证【说明：忽略参数校验可能导致：1.`page size`过大导致内存溢出；2.恶意`order by`导致数据库慢查询；3.任意重定向；4.SQL注入；5.反序列化注入；6.正则输入源串拒绝服务ReDoS；说明：Java代码用正则来验证客户端的输入，有些正则写法验证普通用户输入没有问题，但是如果攻击人员使用的是特殊构造的字符串来验证，有可能导致死循环的结果。】
- 禁止向HTML页面输出未经安全过滤或未正确转义的用户数据
- 表单、AJAX提交必须执行CSRF安全验证【说明：`CSRF(Cross-site request forgery)`跨站请求伪造是一类常见编程漏洞。对于存在CSRF漏洞的应用/网站，攻击者可以事先构造好URL，只要受害者用户一访问，后台便在用户不知情的情况下对数据库中用户参数进行相应修改。】
- 在使用平台资源，譬如短信、邮件、电话、下单、支付，必须实现正确的防重放的 机制，如数量限制、疲劳度控制、验证码校验，避免被滥刷而导致资损
- 发贴、评论、发送即时消息等用户生成内容的场景必须实现防刷、文本内容违禁词过滤等风控策略

## 五、MySQL数据库

### (一) 建表规约
- 表达是与否概念的字段，必须使用`is_xxx`的方式命名，数据类型是`unsigned tinyint`（1表示是，0表示否）【说明：任何字段如果为非负数，必须是unsigned。注意：POJO类中的任何布尔类型的变量，都不要加`is`前缀，所以，需要在`<resultMap>`设置从`is_xxx`到`Xxx`的映射关系。数据库表示是与否的值，使用`tinyint`类型，坚持`is_xxx`的命名方式是为了明确其取值含义与取值范围】
> 正例：表达`逻辑删除`的字段名`is_deleted`，1表示删除，0表示未删除
- 表名、字段名必须使用小写字母或数字，禁止出现数字开头，禁止两个下划线中间只出现数字。数据库字段名的修改代价很大，因为无法进行预发布，所以字段名称需要慎重考虑【说明：MySQL在 Windows下不区分大小写，但在Linux下默认是区分大小写。因此，数据库名、表名、字段名，都不允许出现任何大写字母，避免节外生枝】
> 正例：`aliyun_admin`，`rdc_config`，`level3_name`  
反例：AliyunAdmin，rdcConfig，level_3_name
- 表名不使用复数名词【表名应该仅仅表示表里面的实体内容，不应该表示实体数量，对应于`DO`类名也是单数形式，符合表达习惯】
- 禁用保留字，如`desc、range、match、delayed`等，请参考`MySQL`官方保留字
- 主键索引名为`pk_字段名`；唯一索引名为`uk_字段名`；普通索引名则为`idx_字段名`【说明：`pk_`即`primary key`；`uk_`即`unique key`；`idx_`即`index`的简称】
- 小数类型为`decimal`，禁止使用`float`和`double`【说明：在存储的时候，`float`和`double`都存在精度损失的问题，很可能在比较值的时候，得到不正确的结果。如果存储的数据范围超过`decimal`的范围，建议将数据拆成整数和小数并分开存储】
- 如果存储的字符串长度几乎相等，使用`char`定长字符串类型
- `varchar`是可变长字符串，不预先分配存储空间，长度不要超过`5000`，如果存储长度大于此值，定义字段类型为`text`，独立出来一张表，用主键来对应，避免影响其它字段索引效率
- 表必备三字段：`id, create_time, update_time`【说明：其中`id`必为主键，类型为`bigint unsigned`、单表时自增、步长为1。`create_time, update_time`的类型均为`datetime`类型。】
- 表的命名最好是遵循“业务名称_表的作用”【正例：`alipay_task / force_project / trade_config`】
- 库名与应用名称尽量一致
- 如果修改字段含义或对字段表示的状态追加时，需要及时更新字段注释
- 字段允许适当冗余，以提高查询性能，但必须考虑数据一致【冗余字段应遵循：1）不是频繁修改的字段。2）不是`varchar`超长字段，更不能是`text`字段。3）不是唯一索引的字段】(`正例`：商品类目名称使用频率高，字段长度短，名称基本一不变，可在相关联的表中冗余存储类目名称，避免关联查询)
- 单表行数超过`500万`行或者单表容量超过`2GB`，才推荐进行`分库分表`【说明：如果预计三年后的数据量根本达不到这个级别，请不要在创建表时就分库分表】
- 合适的字符存储长度，不但节约数据库表空间、节约索引存储，更重要的是提升检索速度。如下表，其中无符号值可以避免误存负数，且扩大了表示范围。

|对象| 年龄区间 |类型 |字节| 表示范围|
|---| --- |--- |---|---|
|人|150岁之内|tinyint unsigned| 1|无符号值：0到255|
|龟|数百岁|smallint unsigned |2|无符号值：0到65535|
|恐龙化石|数千万年|int unsigned|4|无符号值：0到约42.9亿|
|太阳|约50亿年|bigint unsigned|8|无符号值：0到约10的19次方|

### (二) 索引规约
- 业务上具有`唯一特性`的字段，即使是多个字段的组合，也必须建成唯一索引【说明：不要以为唯一索引影响了`insert`速度，这个速度损耗可以忽略，但提高查找速度是明显的；另外，即使在应用层做了非常完善的校验控制，只要没有唯一索引，根据墨菲定律，必然有脏数据产生】
- 超过三个表禁止`join`。需要`join`的字段，数据类型必须绝对一致；多表关联查询时，保证被关联的字段需要有`索引`【说明：即使双表join也要注意表索引、SQL性能】
- 在`varchar`字段上建立索引时，必须指定索引长度，没必要对全字段建立索引，根据实际文本区分度决定索引长度即可【说明：索引的长度与区分度是一对矛盾体，一般对字符串类型数据，长度为20的索引，区分度会高达90%以上，可以使用`count(distinct left(列名, 索引长度))/count(*)`的区分度来确定】
- 页面搜索严禁左模糊或者全模糊，如果需要请走搜索引擎来解决【说明：索引文件具有 `B-Tree`的`最左前缀匹配`特性，如果左边的值未确定，那么无法使用此索引】
- 如果有`order by`的场景，请注意利用索引的有序性。`order by`最后的字段是组合索引的一部分，并且放在索引组合顺序的最后，避免出现`file_sort`的情况，影响查询性能
> 正例：where a=? and b=? order by c; 索引：a_b_c  
反例：索引如果存在范围查询，那么索引有序性无法利用，如：WHERE a>10 ORDER BY b; 索引 a_b 无法排序。
- 利用覆盖索引来进行查询操作，避免回表【说明：如果一本书需要知道第11章是什么标题，会翻开第11章对应的那一页吗？目录浏览一下就好，这个目录就是起到覆盖索引的作用。】
> 正例：能够建立索引的种类分为主键索引、唯一索引、普通索引三种，而覆盖索引只是一种查询的一种效果，用`explain`的结果，`extra`列会出现：`using index`
- 利用延迟关联或者子查询优化超多分页场景【说明：MySQL并不是跳过`offset`行，而是取`offset+N`行，然后返回放弃前`offset`行，返回`N`行，那当`offset`特别大的时候，效率就非常的低下，要么控制返回的总页数，要么对超过特定阈值的页数进行SQL改写】
> 正例：先快速定位需要获取的id段，然后再关联：`SELECT a.* FROM 表 1 a, (select id from 表 1 where 条件 LIMIT 100000,20 ) b where a.id=b.id`
- SQL性能优化的目标：至少要达到`range`级别，要求是`ref`级别，如果可以是`consts`最好【说明：1）`consts`单表中最多只有一个匹配行（主键或者唯一索引），在优化阶段即可读取到数据。2）`ref`指的是使用普通的索引（normal index）。3）`range`对索引进行范围检索。】
> 反例：`explain`表的结果，type=index，索引物理文件全扫描，速度非常慢，这个`index`级别比较`range`还低，与全表扫描是小巫见大巫。
- 建组合索引的时候，区分度最高的在最左边【说明：存在非等号和等号混合时，在建索引时，请把等号条件的列前置。如：`where c>? and d=?`那么即使c的区分度更高，也必须把d放在索引的最前列，即索引`idx_d_c`】
> 正例：如果`where a=? and b=?`，如果a列的几乎接近于唯一值，那么只需要单建`idx_a`索引即可
- 防止因字段类型不同造成的隐式转换，导致索引失效
- 创建索引时避免有如下极端误解
> 1）宁滥勿缺。认为一个查询就需要建一个索引  
2）宁缺勿滥。认为索引会消耗空间、严重拖慢记录的更新以及行的新增速度  
3）抵制惟一索引。认为业务的惟一性一律需要在应用层通过“先查后插”方式解决

### (三) SQL语句
- 不要使用`count(列名)`或`count(常量)`来替代`count(*)`，`count(*)`是SQL92定义的标准统计行数的语法，跟数据库无关，跟NULL和非NULL无关【说明：`count(*)`会统计值为`NULL`的行，而`count(列名)`不会统计此列为NULL值的行】
- `count(distinct col)`计算该列除NULL之外的不重复行数，注意`count(distinct col1, col2)`如果其中一列全为 NULL，那么即使另一列有不同的值，也返回为0
- 当某一列的值全是NULL时，`count(col)`的返回结果为 0，但`sum(col)`的返回结果为NULL，因此使用`sum()`时需注意`NPE`问题
> 正例：使用如下方式来避免sum的NPE问题：`SELECT IFNULL(SUM(column), 0) FROM table;`
- 使用`ISNULL()`来判断是否为`NULL`值【说明：NULL与任何值的直接比较都为NULL。 1）`NULL<>NULL`的返回结果是`NULL`，而不是`false`。 2）`NULL=NULL`的返回结果是`NULL`，而不是`true`。 3）`NULL<>1`的返回结果是`NULL`，而不是`true`。】
- 代码中写分页查询逻辑时，若`count`为0应直接返回，避免执行后面的分页语句
- 不得使用外键与级联，一切外键概念必须在应用层解决【说明：以学生和成绩的关系为例，学生表中的`student_id`是主键，那么成绩表中的`student_id`则为外键。如果更新学生表中的`student_id`，同时触发成绩表中的`student_id`更新，即为级联更新。外键与级联更新适用于单机低并发，不适合分布式、高并发集群；级联更新是强阻塞，存在数据库更新风暴的风险；外键影响数据库的插入速度】
- 禁止使用存储过程，存储过程难以调试和扩展，更没有移植性
- 数据订正（特别是删除、修改记录操作）时，要先select，避免出现误删除，确认无误才能执行更新语句
- `in`操作能避免则避免，若实在避免不了，需要仔细评估in后边的集合元素数量，控制在`1000`个之内
- 如果有国际化需要，所有的字符存储与表示，均以`utf-8`编码，注意字符统计函数的区别。【说明：`SELECT LENGTH("轻松工作")；`返回为12；`SELECT CHARACTER_LENGTH("轻松工作")；` 返回为4；如果需要存储表情，那么选择`utf8mb4`来进行存储，注意它与`utf-8`编码的区别。】
- `TRUNCATE TABLE`比`DELETE`速度快，且使用的系统和事务日志资源少，但`TRUNCATE`无事务且不触发`trigger`，有可能造成事故，故不建议在开发代码中使用此语句【说明：`TRUNCATE TABLE`在功能上与不带`WHERE`子句的`DELETE`语句相同】

### (四) ORM映射
- 在表查询中，一律不要使用`*`作为查询的字段列表，需要哪些字段必须明确写明【说明：1）增加查询分析器解析成本。2）增减字段容易与`resultMap`配置不一致。3）无用字段增加网络消耗，尤其是`text`类型的字段】
- POJO类的布尔属性不能加`is`，而数据库字段必须加`is_`，要求在`resultMap`中进行字段与属性之间的映射。【说明：参见定义POJO类以及数据库字段定义规定，在`<resultMap>`中增加映射，是必须的。在`MyBatis Generator`生成的代码中，需要进行对应的修改】
- 不要用`resultClass`当返回参数，即使所有类属性名与数据库字段一一对应，也需要定义；反过来，每一个表也必然有一个`POJO`类与之对应【说明：配置映射关系，使字段与`DO`类解耦，方便维护。】
- `sql.xml`配置参数使用：`#{}，#param#`不要使用`${}`此种方式容易出现SQL 注入
- 不允许直接拿`HashMap`与`Hashtable`作为查询结果集的输出【说明：`resultClass=”Hashtable”`，会置入字段名和属性值，但是值的类型不可控】
- 更新数据表记录时，必须同时更新记录对应的`gmt_modified`字段值为当前时间
- 不要写一个大而全的数据更新接口。传入为POJO类，不管是不是自己的目标更新字段，都进行`update table set c1=value1,c2=value2,c3=value3; `这是不对的。执行SQL时，不要更新无改动的字段，一是易出错；二是效率低；三是增加`binlog`存储
- `@Transactional`事务不要滥用。事务会影响数据库的`QPS`，另外使用事务的地方需要考虑各方面的回滚方案，包括缓存回滚、搜索引擎回滚、消息补偿、统计修正等
- `<isEqual>`中的compareValue是与属性值对比的常量，一般是数字，表示相等时带上此条件；`<isNotEmpty>`表示不为空且不为null时执行；`<isNotNull>`表示不为null值时执行。

## 六、工程结构
- 分层异常处理规约
> 在`DAO`层，产生的异常类型有很多，无法用细粒度的异常进行catch，使用`catch(Exception e)`方式，并`throw new DAOException(e)`，不需要打印日志，因
为日志在`Manager/Service`层一定需要捕获并打印到日志文件中去，如果同台服务器再打日志，浪费性能和存储。在`Service`层出现异常时，必须记录出错日志到磁盘，尽可能带上参数信息，相当于保护案发现场。如果`Manager`层与`Service`同机部署，日志方式与DAO层处理一致，如果是单独部署，则采用与`Service`一致的处理方式。Web 层绝不应该继续往上抛异常，因为已经处于顶层，如果意识到这个异常将导致页面无法正常渲染，那么就应该直接跳转到友好错误页面，加上用户容易理解的错误提示信息。开放接口层要将异常处理成错误码和错误信息方式返回。
- 分层领域模型规约
> `DO`（Data Object）：此对象与数据库表结构一一对应，通过DAO层向上传输数据源对象  
`DTO`（Data Transfer Object）：数据传输对象，Service或Manager向外传输的对象  
`BO`（Business Object）：业务对象，由Service层输出的封装业务逻辑的对象  
`AO`（Application Object）：应用对象，在Web层与Service层之间抽象的复用对象模型，极为贴近展示层，复用度不高  
`VO`（View Object）：显示层对象，通常是Web向模板渲染引擎层传输的对象  
`Query`：数据查询对象，各层接收上层的查询请求。注意超过 2 个参数的查询封装，禁止使用Map类来传输
- 二方库依赖
> 1）`GroupID`格式：`com.{公司/BU }.业务线 [.子业务线]`，最多4级。正例：`com.taobao.jstorm`或`com.alibaba.dubbo.register`  
2）`ArtifactID`格式：`产品线名-模块名`。语义不重复不遗漏，先到中央仓库去查证一下。正例：`dubbo-client / fastjson-api / jstorm-tool`  
3）`Version`：主版本号.次版本号.修订号。【1.主版本号：产品方向改变，或者大规模 API 不兼容，或者架构不兼容升级；2.次版本号：保持相对兼容性，增加主要功能特性，影响范围极小的 API 不兼容修改；3.修订号：保持完全兼容性，修复 BUG、新增次要功能特性等】  
**说明**：注意起始版本号必须为：1.0.0，而不是0.0.1，正式发布的类库必须先去中央仓库进行查证，使版本号有延续性，正式版本号不允许覆盖升级。如当前版本：1.3.3，那么下一个合理的版本号：1.3.4或1.4.0或2.0.0
- 底层基础技术框架、核心数据管理平台、或近硬件端系统谨慎引入第三方实现
- 所有pom文件中的依赖声明放在`<dependencies>`语句块中，所有版本仲裁放在`<dependencyManagement>`语句块中【说明：`<dependencyManagement>`里只是声明版本，并不实现引入，因此子项目需要显式的声明依赖，`version`和`scope`都读取自父pom。而`<dependencies>`所有声明在主pom的`<dependencies>`里的依赖都会自动引入，并默认被所有的子项目继承】
- 二方库不要有配置项，最低限度不要再增加配置项
- 为避免应用二方库的依赖冲突问题，二方库发布者应当遵循以下原则
> 1）精简可控原则。移除一切不必要的API和依赖，只包含Service API、必要的领域模型对象、Utils类、常量、枚举等  
2）稳定可追溯原则。每个版本的变化应该被记录，二方库由谁维护，源码在哪里，都需要能方便查到
- 高并发服务器建议调小`TCP`协议的`time_wait`超时时间【操作系统默认`240`秒后，才会关闭处于`time_wait`状态的连接，在高并发访问下，服务器端会因为处于`time_wait`的连接数太多，可能无法建立新的连接，所以需要在服务器上调小此等待值】
> 正例：在linux服务器上请通过变更`/etc/sysctl.conf`文件去修改该缺省值（秒）：`net.ipv4.tcp_fin_timeout = 30`
- 调大服务器所支持的最大文件句柄数（`File Descriptor`，简写为 fd）【说明：主流操作系统的设计是将`TCP/UDP`连接采用与文件一样的方式去管理，即一个连接对应于一个fd。主流的linux服务器默认所支持最大fd数量为`1024`，当并发连接数很大时很容易因为fd不足而出现`“open too many files”`错误，导致新的连接无法建立。建议将linux服务器所支持的最大句柄数调高数倍（与服务器的内存数量相关）】
- 给JVM环境参数设置`-XX:+HeapDumpOnOutOfMemoryError`参数，让`JVM`碰到
  `OOM`场景时输出`dump`信息【说明：OOM的发生是有概率的，甚至相隔数月才出现一例，出错时的堆内信息对解决问题非常有帮助】
- 在线上生产环境，JVM的Xms和Xmx设置一样大小的内存容量，避免在GC后调整堆大小带来的压力

## 七、设计规约
- 存储方案和数据结构需要认真地进行设计和评审
- 需求分析与系统设计在考虑主干功能的同时，需要充分评估异常流程与业务边界
- 谨慎使用继承的方式来进行扩展，优先使用聚合/组合的方式来实现
- 系统设计主要目的是明确需求、理顺逻辑、后期维护，次要目的用于指导编码。
- 设计的本质就是识别和表达系统难点，找到系统的变化点，并隔离变化点
- 系统架构设计的目的【1.确定系统边界。确定系统在技术层面上的做与不做；2.确定系统内模块之间的关系。确定模块之间的依赖关系及模块的宏观输入与输出； 3.确定指导后续设计与演化的原则。使后续的子系统或模块设计在规定的框架内继续演化；4.确定非功能性需求。非功能性需求是指安全性、可用性、可扩展性等】

## 个人小结
以上内容便是我一边读，一边思考整理出来的。《Java开发手册》是实践开发中提炼出来的`精华`，当中有很多小点都是值得拿出来仔细推敲，往往小的规约背后隐藏着大学问。或许你和我一样，有些地方并不熟悉或者暂时并不理解为什么这样规约，个人觉得这些困惑的点读一遍、思考一遍远远不够，应该收藏下来，有空的时候多读、多思考。如果开发中遇到了手册中类似的场景，那么收获会更大，理解会更深。最后，正如手册中提到的愿景，“码出高效，码出质量”，希望你我都能码出一个新的高度。