## LogMinds
一些关于日志打印的思考和实践总结、日志规范

## 目录
```
 版本:1.8
 最新更新日期：2022/04/19
```

```
一、【实施目标】
二、【日志的副作用】
三、【打印规范】
四、【日志禁忌】
五、【日志实践】
六、【日志文件规范】
七、【日志配套解决方案】
八、【附件】
```

## 日志规范三原则
打印的时候，遵循以下**三个原则**：
- 必要：打印的位置恰当。
- 有效：打印的内容有用。
- 最小：不要重复打印类似的内容。

>我们可以通过以下步骤来实现上述三个原则：
1、日志打印前，要了解【副作用】，确定是否打印。
2、日志打印中，要依据【打印规范】【日志禁忌】【日志实践】进行正确打印。
3、日志打印后，要根据【日志文件规范】对日志文件进行清理。
4、最后，参考【日志配套解决方案】选择想要的功能进行接入，推荐优先接入日志平台，统一集中管理和查看，同时为云原生做准备。

## 一、【实施目标】
&ensp;&ensp;&ensp;&ensp;本文主要着重于解决日志打印中常见的各类问题，提供一个基本的规范、场景指引和实践参考，主要聚焦以下三个目标：
- 提高日志定位问题的效率；
- 提高日志打印性能；
- 降低日志运维成本。

> 本文多数内容主要针对Java技术栈下的日志打印。

## 二、【副作用】
&ensp;&ensp;&ensp;&ensp;首先，对于日志的问题，我们首先要了解一些副作用，这样才能在我们要打日志的时候，根据实际情况进行取舍。

### 2.1 性能损耗
&ensp;&ensp;&ensp;&ensp;对于日志来说，根据实际情况测试，日志对性能的影响较大。虽然随着日志库的升级，会有一些缓解，但是总体还是对性能影响较大。  
&ensp;&ensp;&ensp;&ensp;但是，一般情况下终极的性能并非我们首要考虑的情况，所以为了日常问题查询和发现等，必要日志打印必不可少。只要性能测试符合预期，那么正常打印即可，否则可以考虑优化日志打印内容和数量。

### 2.2 存储占用
&ensp;&ensp;&ensp;&ensp;由于日志需要占用存储，所以一定需要配套的清理策略，而且某些异常下导致程序死循环的情况会瞬间拉爆服务器导致服务不可用。

### 2.3 信息泄露
&ensp;&ensp;&ensp;&ensp;打印的敏感信息被明文存储于文件中，一旦系统发生漏洞或者管理不当，则容易发生信息泄露。

### 2.4 查找难度
&ensp;&ensp;&ensp;&ensp;不规范的日志打印太多，增加了查找关键日志信息的检索难度。

## 三、【打印规范】
### 3.1 配置规范
#### 3.1.1 打印格式
[日志级别|线程号|时间|类名:行号|日志内容]

- Log4j2的LOG_PATTERN：
`[%.4p|%.10t|%d{HH:mm:ss.SSS} %c{1}:%L|%m]%n`

- 样例：
`[INFO|080-exec-3|10:11:28.256|LoginCtrl:68|假装这是登录的第一个业务日志.]`

#### 3.1.2 日志级别
- 优先级
> All < TRACE < DEBUG < **INFO < WARN < ERROR < FATAL < OFF**

- 用法

级别 | 中文 | 用法
---|---|---
DEBUG | 调试 | 指明细致的事件信息，对调试应用最有用。
INFO | 信息 | 指明描述信息，从粗粒度上描述了应用运行过程。
ERROR | 错误 | 指明错误事件，但应用很小可能还能继续运行。
> 生产日志至少是INFO级别，DEBUG的日志如非必要则不能打印到生产。
> 对于一般的应用系统来讲，主要用到的有DEBUG、INFO、和ERROR三个级别的日志即可。

#### 3.1.3 关闭重复打印
避免重复（父子类以及根类）打印日志，浪费磁盘空间，务必在日志配置文件中设置 additivity=false
```
 <Logger name="bcs.logtest" level="info" additivity="false"><AppenderRef ref="applog_asyn" /></Logger> 
```

### 3.2 错误日志打印规范
&ensp;&ensp;&ensp;&ensp;所有日志打印的内容要能体现以下两个关键点：
- **位置**：即当前程序调用处于调用流程中的哪个节点。
- **数据**：当前节点的状态数据（请求、响应和用户数据）是什么。 

#### 3.2.1 error日志
&ensp;&ensp;&ensp;&ensp;对于**任何出错返回**的情况，都要打印error日志，同时要求打印输入参数、返回的结果数据、如果能获取客户信息则也要一起打印。
> 错误日志可以尽量详细的打印，因为一个正常的系统，出错的情况应该是少数情况，详细的日志打印有利于快速定位问题，同时不会造成太多的系统运行和运维负担。

#### 3.2.2 异常日志
&ensp;&ensp;&ensp;&ensp; 同时对于捕获的异常错误，即Java中的异常类（Exception等），需要打印调用链路的信息，可以快速定位到异常位置和调用路径，同时需要在**异常抛出点额外打印状态数据**或者**在调用前打印输入参数**。

> 其实，比较理想的错误日志处理是通过异常返回错误信息，这样可以更简单清晰的知道调用路径。

### 3.3 关键路径日志规范
&ensp;&ensp;&ensp;&ensp;关键路径日志，即交易的主要流程可以通过这些日志将交易的路径清晰的展现出来。

#### 3.3.1 关键路径日志作用
&ensp;&ensp;&ensp;&ensp;一般是作为错误日志打印的一个补充，这个主要在查错的时候，有以下两个作用：
- 通过关键路径日志确定发生错误时的调用路径和各路径节点的状态数据，更容易找出问题的原因所在。
- 在某些错误日志漏打的情况下，我们能通过关键路径日志缩小错误定位的排查范围。

&ensp;&ensp;&ensp;&ensp;我们以转账为例（**假设在一个系统内完成**）：
![Image](https://github.com/HiRobot/LogMinds/blob/main/example-transfer.png)
&ensp;&ensp;&ensp;&ensp;如上图所示，如果在**营销通知**环节出了问题，我们可能需要知道是走哪条路径来到的营销通知环节，方便分析和定位问题，关键路径日志打印主要有以下两个方法。

#### 3.3.1 关键路径日志打印方法1-入口参数法
&ensp;&ensp;&ensp;&ensp;由于程序是对数据的处理，不同调用中变化的是数据，所以我们如果能获取到所有的参数数据，则一般情况下都可以从代码中推导出程序的关键路径。
- 优点：统一处理，简单。
- 缺点：推断耗时，效率不高。

#### 3.3.2 关键路径日志打印方法2-关键节点打印法
&ensp;&ensp;&ensp;&ensp;在关键的功能节点打印出当前的关联的状态数据，要求能体现当前节点的所有状态，一般是输入到当前节点的必要参数和**客户信息**（用于做关联），也可以根据实际需要打印输出数据。
- 优点：日志清晰展示了程序运行流程，精确的体现了当前程序的状态。
- 缺点：存在代码编写成本，而且可能带来大量的日志。

#### 3.3.3 关键路径日志实践
- 一般情况下是两种结合使用，对于容易出错或者黑盒调用则采用关键节点打印的方法。
> 黑盒调用：即无法通过代码知道逻辑的调用，例如调用jar包中的方法或者和调用其他系统接口。

### 四、【日志禁忌】
### 4.1 禁止标准输出打印
&ensp;&ensp;&ensp;&ensp;生产禁止将日志打印到标准输出和标准错误输出，由于标准输出没有缓存，打印未做任何优化，极大影响程序性能。  
&ensp;&ensp;&ensp;&ensp;一般情况下，应用日志库会将标准输出日志单独打印到一个文件中，例如Web容器的日志文件（像Tomcat的catalina.out中），不会和log4j等的应用日志一起打印，这样很不利于错误的排查。  
&ensp;&ensp;&ensp;&ensp;同时，容器的日志一般需要运维手工备份或者切割，提高了运维成本。
```
错误示例:
   System.out.println("进展顺利");
   System.err.println("发生错误")

正确示例：
    slf4jLogger.info("进展顺利");
    slf4jLogger.error("发生错误");
```

### 4.2 禁止异常调用printStackTrace()
&ensp;&ensp;&ensp;&ensp;该异常打印习惯主要有以下两个重大缺点：
- Throwable的printStackTrace()会将异常栈打印到标准错误输出中，所以该调用违反了【4.1 禁止打印到标准输出】;
- 同时该打印方法丢失了线程和时间信息，无法和对应交易进行关联，极大的提高了日志的分析难度。 

**正确的异常栈打印方法如下**：
```
	try
	{
		doTransfer();
	}
	catch ( Exception e )
	{
	    // 错误示例：IDE自动补全的该方法。
		e.printStackTrace();
		
		// 正确示例：可以正确打印调用栈信息到合适的日志文件中。
		slf4jLogger.error("转账异常:"+e.getMessage(),e);
	}
```

### 4.3 禁止分行打印
&ensp;&ensp;&ensp;&ensp;如非必要，==单次日志打印调用==需要打印在一行，不建议换行，因为一行日志一般是一个完备的内容。分行后，主要有以下问题：
```
1、使用一般的grep脚本比较难以查看，需要打开日志文件寻找，很不方便。  
2、分行打印后，换行后打印的日志中，线程、时间信息将丢失，同时日志量较大的情况下会出现混合打印的情况，无法分辨出是哪个交易的日志。
```
&ensp;&ensp;&ensp;&ensp;如果是为了方便查看的话，可以使用工具将xml格式化。
```
linux环境下格式化输出命令:
    xmllint --format yourfile.xml

windows环境下：
    UE也可以格式化
```

### 4.4 禁止打印在工程目录内部
&ensp;&ensp;&ensp;&ensp;日志打印目录不能设置在web工程内部，这样程序部署的时候可能会删除日志，也不利于日常的日志维护。
```
    Log4j2的配置建议：
        <property name="LOG_HOME">${sys:user.home}/__logs/你的应用目录名字</property>
```

### 4.5 不要打印媒体数据
&ensp;&ensp;&ensp;&ensp;在info级别以下不要打印媒体数据，debug级别可以根据调试情况打印媒体数据；
&ensp;&ensp;&ensp;&ensp;生产环境下，对于图片、音频、视频等多媒体数据做了base64的后的字符串，建议进行媒体数据去除后处理后再打印。
```
    当前并未有特别好的方法进行控制：
        1、依赖开发人员的不要主动打印媒体数据。
        2、对于底层框架层的话，需要根据开发人员根据配置进行媒体内容去除后打印。
```

## 五、【日志实践】
&ensp;&ensp;&ensp;&ensp;日志打印的时候，我们需要有面向生产级规模日志打印的意识，生产日志具备两个重要的特征：
- 【**量大**】，不是像测试环境一样，靠简单的翻阅就能找到目标日志，生产是找不到的，计算看着像也不一定是对的。
- 【**紧急**】，需要在有限的时间内快速检索到正确的目标日志，特别是投产查询问题的时候，是要求极短的时间内能快速定位问题。一般采用关联打印的方法。

### 5.1 关联打印
&ensp;&ensp;&ensp;&ensp;日志内容打印，建议和客户信息进行关联，因为一般问题的定位的时候会很容易的获得客户信息，但不一定能获得各类交易信息。
&ensp;&ensp;&ensp;&ensp;例如，你给客户生成了一个贷款的电子合同，把文件名打印到了日志中，这样出问题的时候，这个日志在测试环境中可能可以用，就去找最近一笔就行了。
&ensp;&ensp;&ensp;&ensp;但是到生产环境中，这类交易会很多，无法按照最近一笔去定位。所以最好的习惯就是将客户手机号（或这客户号或者姓名等）+文件名进行关联打印到日志中；当业务人员告诉你XX客户签订合同报错的时候，你按照客户号搜索即可找到对应信息。
&ensp;&ensp;&ensp;&ensp;实践方法总结：
- 和客户信息关联
- 和业务场景关联
- 和以上两者同时关联。
```
    反例：
        logger.info("{}",filename);
        
    正例：
        logger.info("秒贷电子合同:{}",filename);
        logger.info("客户:{},秒贷电子合同:{}",userName,filename);
```

### 5.2 slf4j解耦
&ensp;&ensp;&ensp;&ensp;一般建议使用slf4j结合log4j或者logback进行使用，主要可以获得以下两个好处：
- 对具体的日志依赖实现进行解耦，方便后续的日志实现升级和更换。
- 使用占位符进行内容格式化，用以提高内存使用效率。
```
   slf4jLogger.info("客户{}使用卡{}进行转账", userName, cardNo );
```

>写法比较
```
    1、常规写法：性能（内存）损耗。
        log4jLogger.debug("客户"+userName+"使用卡"+cardNo+"进行转账");
    	log4jLogger.debug(String.format("客户%s使用卡%s进行转账", userName, cardNo ));
    	
    2、优化写法：显得累赘。
        if( log4jLogger.isDebugEnabled() )
		{
		    log4jLogger.debug(String.format("客户%s使用卡%s进行转账", userName, cardNo ));
		}
		
	3、优雅的写法：
	     slf4jLogger.debug("客户{}使用卡{}进行转账", userName, cardNo );
```

### 5.3 字符串拼接
&ensp;&ensp;&ensp;&ensp;当拥有大量字符串（超过4个）进行拼接的时候，使用StringBuilder进行拼接，不要使用+号进行直接拼接。
```
    错误示例:
        String errMsg = "这是"+"一个错误的示例"+"不要学"+"这个糟糕的写法";
        
    正确示例：
        String errMsg = new StringBuilder(64)
			.append("这是")
			.append("一个正确是写法")
			.append("请习惯这个写法")
			.append("而且append方法特别好用").toString();
```

### 5.4 那么到底如何打印
&ensp;&ensp;&ensp;&ensp;假象一下上面的转账流程，如果没有任何日志，当要你查找一个转账出错的问题，是否可以感受到那种深深的、无力的绝望；这种无力感就是大多数时候就是（项目经理或者运维人员）查询日常工单时，查看日志时发现无从下手的感觉，然后瞬间变为心中万马奔腾的感觉。
&ensp;&ensp;&ensp;&ensp;所以，在编写完代码的后，可以自己跳出来想想，在生产几十万甚至几千万条日志的情况下，如何才能快速精确的定位到可能出现的问题，同时不对系统运行和运维带来沉重的负担。
&ensp;&ensp;&ensp;&ensp;详见【**8.1 代码样例**】

---

## 六、日志文件规范
### 6.1 日志文件类型
&ensp;&ensp;&ensp;&ensp;一般应用系统日志的日志由应用日志和容器日志两种日志文件类型，一般情况下很容易漏掉容器日志文件的管理。
#### 6.1.1 应用日志文件
&ensp;&ensp;&ensp;&ensp;即所用日志SDK定义的日志，比如log4j，logback等日志库配置的日志文件。

#### 6.1.2 容器日志文件
&ensp;&ensp;&ensp;&ensp;即容器自带的文件，像Tomcat容器中logs下面的catalina.out文件。

### 6.2 日志命名
&ensp;&ensp;&ensp;&ensp;使用“应用名.log”来保存（一般为工程名字），保存在$HOME/__logs/{应用名}/logs/目录下，历史日志命名一般为{logname}.log.{保存日期}。

### 6.3 日志文件大小
&ensp;&ensp;&ensp;&ensp;一般一个日志文件2G左右比较合适，最大不能超过4G。因为有些情况下可能需要手工打开日志文件，太大则容易挤爆服务器。  
&ensp;&ensp;&ensp;&ensp;同时对于Web容器下的日志，需要按天或者小时进行分隔。

#### 6.3.1 应用日志文件大小配置
```
对于使用log4j2的日志库的，配置Log4j2的<SizeBasedTriggeringPolicy size="2G" />可以进行日志大小配置。
```

#### 6.3.2 容器日志文件大小配置
```
Web容器下，没有自带日志切割功能的，可以使用cronolog按照天或者小时进行切割。
```

### 6.4 日志生命周期管理
&ensp;&ensp;&ensp;&ensp;在系统投产时，就需要确定日志（包含应用日志和容器日志）的备份和清理策略，一般是本地保留最近两周的日志，两周以外的备份到远程日志服务器并删除本地日志文件。可根据系统灵活制定清理策略，但是一定要有对应的清理动作。

### 6.5 日志文件查看
&ensp;&ensp;&ensp;&ensp;如果需要本地打开日志则一般是less打开节省内存，而非vi或者view（会尝试将日志全部装在到内存中）导致内存拉爆。

## 七、【日志配套解决方案】

### 7.1 日志平台
&ensp;&ensp;&ensp;&ensp;主要解决在应用集群中搜索日志困难的问题，提供统一的查询终端（界面）进行查询。

#### 7.1.1 状态
&ensp;&ensp;&ensp;&ensp;可用，硬件扩容中。

#### 7.1.2 对接人
&ensp;&ensp;&ensp;&ensp;黄悦

---

### 7.2 客户事件日志
&ensp;&ensp;&ensp;&ensp;可根据客户、时间、事件和结果四要素进行查询，对一般问题提供了统一的查询解决方案，解决了日常日志查询中经常要根据场景分析如何搜索的情况。
> 例如转账问题可能是按照卡号搜索，贷款可能是按照身份证号码搜，这类条件搜索需要针对每个场景进行现场分析，而且卡号和身份证号会出现在其他的业务场景中，需要人工继续分析排查，在生产那样量级的日志中，难度可想而知。

#### 7.2.1 状态
&ensp;&ensp;&ensp;&ensp;可用，依赖日志平台。

#### 7.2.2 对接人
&ensp;&ensp;&ensp;&ensp;张超晴

---

### 7.3 日志脱敏
&ensp;&ensp;&ensp;&ensp;用于解决敏感信息打印的问题。

#### 7.3.1 状态
&ensp;&ensp;&ensp;&ensp;可用。

#### 7.3.2 对接人
&ensp;&ensp;&ensp;&ensp;张超晴

---

### 7.4 服务异常预警
&ensp;&ensp;&ensp;&ensp;基于【客户事件日志】中的交易信息，进行实时计算出日志事件指标，从而对达到一定阈值的事件进行自动预警。

#### 7.4.1 状态
&ensp;&ensp;&ensp;&ensp;试运行阶段

---

### 7.5 统一客户状态流水号
&ensp;&ensp;&ensp;&ensp;即带客户状态的流水号日志，用于解决后端业务系统无法从平台层统一感知客户的问题，同时和全局流水号类似，用于解决各个系统之间没有关联的系统级流水号的问题。
> 后端业务系统不像前端渠道系统，没有维护了客户的登录状态，没有统一的方法知道当前每笔交易是谁在做操作。

#### 7.5.1 状态
&ensp;&ensp;&ensp;&ensp;建设中，方案已完成。

---

### 7.6 日志治理
&ensp;&ensp;&ensp;&ensp;用于排查哪些系统在单个请求（交易）中打印了过多日志的问题。

#### 7.6.1 状态
&ensp;&ensp;&ensp;&ensp;建设中，方案制定中。

#### 7.6.2 单行治理
&ensp;&ensp;&ensp;&ensp;根据日志平台采集内容，对个应用平台日志进行分析统计，找出日志单条打印较大的日志进行分析和优化。
```
log4j2中加入以下配置可以限制长度：%maxLen{%m}{log-length}%n
    <property name="LOG_PATTERN">[%.4p|%.10t|%d{HH:mm:ss.SSS} %c{1}:%L|%maxLen{%m}{512}%n</property>
```

#### 7.6.3 单次交易治理
&ensp;&ensp;&ensp;&ensp;根据日志平台采集内容，对个应用平台日志进行分析统计，找出单次请求打印日志的较大的交易进行分析和优化。

## 八、附件
### 8.1 代码样例
#### 8.1.1 反例
```
    // 反例：如果转账返回false，则不知道是在哪一布返回的。
	// 转账模块
	public boolean doTransfer( TransferReqModel reqModel )
	{
		// 信息校验
		if ( !checkTransferInfo(reqModel) )
		{
			return false;
		}
		
		// 风控校验
		if ( !checkTransferRisk(reqModel) )
		{
			return false;
		}
		
		// 执行转账
		if ( !execTransfer(reqModel) )
		{
			return false;
		}
		
		return true;
	}
```

#### 8.1.2 正例
```
    // 正例：如果false，则可以知道是在哪一步返回的，且可以知道传入的参数是什么。
	// 转账模块
	public boolean doTransfer( TransferReqModel reqmodel )
	{
	    // 关键路径打印：保留业务场景数据。
		slf4jLogger.info("{}转账:{}",userInfo.getUserId(),JsonCoder.toString(reqmodel));
		
		// 信息校验
		if ( !checkTransferInfo(reqmodel) )
		{
		    // 错误日志打印：标记错误返回位置和信息。
			slf4jLogger.error("转账,信息校验失败,userId={}",userId);
			return false;
		}
		
		// 风控校验
		RiskReqModel  riskModel = ConvertData.convert( userInfo, reqmodel);  // 转换bean
		if ( !checkTransferRisk(riskModel) )
		{
			slf4jLogger.error("转账,风控检查失败,,userId={}",userId);
			return false;
		}
		
		// 执行转账
		if ( !execTransfer(reqmodel) )
		{
			slf4jLogger.error("转账,执行失败,,userId={}",userId);
			return false;
		}
		
		slf4jLogger.info("转账,执行成功,,userId={}",userId);
		return true;
	}
	
	// 风控校验:黑盒调用
	public boolean checkTransferRisk( RiskReqModel reqModel )
	{
	    // 调用风控接口
	    // 关键路径打印：已包含用户信息。
	    slf4jLogger.info("风控检查-开始,userId={},data={}",
	               reqModel.getUserId(),JsonCoder.toString(reqModel));
	    RiskRspModel rspModel = callRiskService(reqModel);      // 黑盒调用风控接口。
	    
	    // 风控校验失败
	    if ( !rspModel.isSuccess() )
	    {
	        // 错误日志打印
	        slf4jLogger.error("风控检查-失败:userId={},errmsg={}",
	                       reqModel.getUserId(),rspModel.getErrorMessage());
	        return false;
	    }
	    
	     slf4jLogger.info("风控检查-完成,userId={}",reqModel.getUserId());
	    return true;
	}
```
