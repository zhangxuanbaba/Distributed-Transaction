#### 分布式事务之说说TCC事务

##### 一、Tcc机制
TCC是三个英文单词的首字母缩写而来。没错，TCC分别对应Try、Confirm和Cancel三种操作，这三种操作的业务含义如下：

1、Try(初步操作)

	尝试执行业务
	完成所有业务检查(一致性)
	预留必须业务资源(准隔离性)

2、Confirm（确认操作）

	确认执行业务
	真正执行业务
	不做任何业务检查，只使用Try阶段预留的业务资源

3、Cancel（取消操作）

	取消执行业务
	释放Try阶段预留的业务资源

其事务处理方式为：

	在全局事务决定提交时，调用与try业务逻辑相对应的confirm业务逻辑；
	在全局事务决定回滚时，调用与try业务逻辑相对应的cancel业务逻辑。

##### 二、TCC优缺点

 TCC事务的优点如下：

	解决了跨应用业务操作的原子性问题，在诸如组合支付、账务拆分场景非常实用。
	TCC实际上把数据库层的二阶段提交上提到了应用层来实现，对于数据库来说是一阶段提交，规避了数据库层的2PC性能低下问题。

TCC事务的缺点，主要就一个：

	TCC的Try、Confirm和Cancel操作功能需业务提供，开发成本高。
	需要写很多的补偿代码去弥补程序错误可能造成的问题，代码量较多。

当然，对TCC事务的这个缺点是否是缺点，是一个见仁见智的事情。

##### 三、如何理解分布式事务Tcc
以下两篇博客说的比较全面，我就不copy了，大家自己看下吧，详细介绍了一下tcc的原理

	参考博客：
	http://www.iigrowing.cn/ru_he_li_jie_tcc_fen_bu_shi_shi_wu.html
	https://blog.csdn.net/u010412301/article/details/78410933

##### 四、TCC分布式事务实战
终于进入正题了哦，开始前，先给大家说明一下哈，因为目前开源的tcc分布式事务框架并不多，博主在找了许久之后，亲自测试了tcc-transaction,bytetcc,hmily这三种tcc框架，都是在github上开源的，最终选定了tcc-transaction框架，也是目前博主找到的github上star最多的一个项目。

	tcc-transaction 使用指南1.2.x
	https://github.com/changmingxie/tcc-transaction/wiki/%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%971.2.x

	推荐一个博客：https://blog.csdn.net/l1028386804/article/details/73731363
	这个博客写的相当全面，从生成jar到实战都有涉及，但是可能因为博主写的比较急，有一些坑没有说明清楚，下面我将基于这个博客，把博主碰到的坑给大家讲解一下。

博主使用的是dubbo项目

###### 1.配置文件pom.xml
 必须要引入tcc-transaction-spring和tcc-transaction-dubbo两个jar包，提供方和调用方都必须引入

###### 2.配置文件spring-context.xml
	 1）必须要配置tcc自己的数据源，源数据源不做修改

	 2)必须引入 <import resource="classpath:tcc-transaction.xml" />

	 3）如果配置了 <aop:aspectj-autoproxy/>，那必须要去掉，因为在tcc-transaction.xml中配置了 <aop:aspectj-autoproxy proxy-target-class="true"/>，只有aop被设置成基类代理才能被tcc框架切入使用

	  proxy-target-class属性值决定是基于接口的还是基于类的代理被创建。如果proxy-target-class 属性值被设置为true，那么基于类的代理将起作用（这时需要cglib库）。如果proxy-target-class属值被设置为false或者这个属性被省略，那么标准的JDK 基于接口的代理

下面是楼主的配置文件

		<bean id="tccDataSource" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
			<property name="driverClassName" value="${jdbc.driver}" />
			<property name="url" value="${jdbc.url}" />
			<property name="username" value="${jdbc.username}" />
			<property name="password" value="${jdbc.password}" />
			<property name="defaultAutoCommit" value="true" />

		</bean>

		<bean id="transactionRepository"
			  class="org.mengyun.tcctransaction.spring.repository.SpringJdbcTransactionRepository">
			<property name="dataSource" ref="tccDataSource"/>
			<property name="domain" value="SAAS"/>
			<property name="tbSuffix" value="_ASSET"/>
		</bean>

		<bean class="org.mengyun.tcctransaction.spring.recover.DefaultRecoverConfig">
			<property name="maxRetryCount" value="30"/>
			<property name="recoverDuration" value="120"/>
			<property name="cronExpression" value="0 */1 * * * ?"/>
			<property name="delayCancelExceptions">
				<util:set>
					<value>com.alibaba.dubbo.remoting.TimeoutException</value>
				</util:set>
			</property>
		</bean>

	 <import resource="classpath:tcc-transaction.xml" />

###### 3.代码编写
tcc transaction框架 有try confirm cancel三个方法，所以在编写代码的时候必须要写满三个接口，并在impl实现的时候也要写上三个方法的逻辑

@Compensable注解以及@Transactional注解必须要要加

具体代码如下

	@Path("/tccOneFacade")
	@Consumes({MediaType.APPLICATION_JSON, MediaType.TEXT_XML})
	@Produces({ContentType.APPLICATION_JSON_UTF_8, ContentType.TEXT_XML_UTF_8})
	public interface TccOneFacade {

		@Path("/insertTccOne")
		@Compensable
		public void insertTccOne(JSONObject object);

		@Path("/cancelTccOne")
		public void cancelTccOne(JSONObject object);

		@Path("/confirmTccOne")
		public void confirmTccOne(JSONObject object);
	}




	@Service
	@Component
	@Transactional(rollbackFor = Exception.class)
	@com.alibaba.dubbo.config.annotation.Service(protocol = {"dubbo"})
	public class TccOneService implements TccOneFacade {


		@Autowired
		private TestTccDao testTccDao;


		@Override
		@Compensable(confirmMethod = "confirmTccOne", cancelMethod = "cancelTccOne", transactionContextEditor = DubboTransactionContextEditor.class)
		@Transactional(propagation = Propagation.REQUIRED, rollbackFor = { Exception.class })
		public void insertTccOne(JSONObject object){
			try {
				this.testTccDao.insertTccOne(object);
				this.testTccDao.insertTccTwo(object);
			} catch (Exception e) {
				throw new RuntimeException(e);
			}
		}

		@Override
		@Transactional(propagation = Propagation.REQUIRED, rollbackFor = { Exception.class })
		public void cancelTccOne(JSONObject object){
				System.err.println("==================新增第一条数据失败，进入cancel阶段===============================");
		}

		@Override
		@Transactional(propagation = Propagation.REQUIRED, rollbackFor = { Exception.class })
		public void confirmTccOne(JSONObject object){
				System.err.println("==================新增第一条数据成功，进入confirm阶段===============================");
		}
	}


###### 4.tcc原理回顾
下面说的是一个困扰了博主一个星期的问题额

tcc原理必须要知道，try、confirm和cancel，博主当时就是没有完全弄清原理，所以被困扰了很久。

比如说，在try中调用a和b两个方法，新增两条数据，a成功，b失败。
一般来说，我们会认为他们会直接回滚，两条数据都不添加。

然而在tcc框架中，他们失败之后会进入cancel方法，这个时候，实际上try方法里面已经commit成功了，也就说第一条数据插入成功，第二条数据插入失败。

这个时候我们有两个选择：
1.将第一条插入的数据删除。
2.将第二条插入失败的数据再次插入。
这个就是tcc的事务补偿机制。

如果在confirm和cancel的时候，方法还是失败了，那么会在数据库中有一个记录，然后会每隔一两分钟就会重新执行cancel方法或者confirm方法，直到方法成功才会停止，或者人工去解决问题。

这也就要求了我们在写confirm和cancel方法的时候必须要满足幂等性，做好充分的判断，保证事务的一致性，所以在客观上增加了开发的成本，需要写很多补偿代码。
