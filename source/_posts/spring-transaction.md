## Transaction注解
spring声明式事务有多种方式,使用注解相对来说更加灵活,使用方式如下:  
1.配置文件中添加:<tx:annotation-driven transaction-manager="transactionManager"/>  
自定义namespaceHandler使用AnnotationDrivenBeanDefinitionParser添加InfrastructureAdvisorAutoProxyCreator到bean容器中,该bean是一个BeanPostProcessor 

2.1 注册BeanFactoryTransactionAttributeSourceAdvisor,该类是用于增强事务的Advisor   
2.2 注册AnnotationTransactionAttributeSource用于2.3解析调用方法的事务注解属性,返回TransanctionAttribute-->txAttr  
2.3 注册TransactionInterceptor作为2.1 Advisor的Advise,用来实现事务,会注入transactionManager和transactionAttributeSource    

3.当从容器中获取bean时,查找所有的Advisor,然后判断步骤1注册的Advisor是否可以应用于该bean,步骤1注册的Advisor采用TransactionAttributeSourcePointcut  
作为PointCut,它是一个StaticMethodPointCut,ClassFilter为TRUE,而方法Matcher的matches,采用判断目标方法或者类上是否有事务注解,返回txAttr  

4.返回匹配的Advisor,然后创建代理对象,使用Advisor作为拦截器实现事务功能,如果一个类或者方法没有使用事务注解,那么不会返回事务Advisor  

 
5.当调用事务增强的方法时,首先执行TransactionInterceptor.invoke(...),阅读代码    
~~~
	protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
        //在判断是否创建代理时,pointCut中方法methodMatche.matches已经解析过事务属性了,这里直接返回
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
        //解析<annotaion-driven>注解时注册到TransactoinInterceptor的TM
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
            //判断是否创建事务,
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
        ...//还有代码
            }
~~~

~~~
	protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
        //如果有txAttr(方法或类上使用了事务注解)并且有事务管理器,创建TransactionStatus
        //事务状态对象包含了事务执行状态和savepoint信息
        //如果txAttr和tm都不为空,就要创建事务对象txObject(包含事务连接),为执行事务做准备了,
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +
							"] because no transaction manager has been configured");
				}
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
~~~

~~~
//准备获取事务,处理事务传播特性
@Override
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
		Object transaction = doGetTransaction();//返回txObject

		// Cache debug flag to avoid repeated checks.
		boolean debugEnabled = logger.isDebugEnabled();

		if (definition == null) {
			// Use defaults if no transaction definition given.
			definition = new DefaultTransactionDefinition();
		}

		if (isExistingTransaction(transaction)) {
            //如果txObject的的connectionHolder为空,说明没有事务,如果不为空并且是active(获取连接是设置为true)那么存在事务,要处理事务传播特性
			// Existing transaction found -> check propagation behavior to find out how to behave.
            // 新的事务方法调用会返回新的事务状态对象,包含方法的事务属性,txObject
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// No existing transaction found -> check propagation behavior to find out how to proceed.
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
                    //当前线程还没有事务,准备创建事务,因为还没有事务所以被挂起的事务为null
                    //事务挂起类似中断处理,将当前事务状态保存,并清除事务同步管理器的状态
			SuspendedResourcesHolder suspendedResources = suspend(null);
			if (debugEnabled) {
				logger.debug("Creating new transaction with name [" + definition.getName() + "]: " + definition);
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				//从数据库获取连接,为txObject设置connHolder,
                //绑定connHolder到事务同步管理器,后面的方法可以通过datasource获取
                //TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
                doBegin(transaction, definition);
                //事务状态对象包含了事务txAttr,事务对象被挂起的事务(用于恢复被挂起的事务)
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException ex) {
				resume(null, suspendedResources);
				throw ex;
			}
			catch (Error err) {
				resume(null, suspendedResources);
				throw err;
			}
		}
		else {
			// Create "empty" transaction: no actual transaction, but potentially synchronization.
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
~~~



## 事务传播实现
1.通过TM的数据源为key,查找connHolder  
2.connHolder如果没有,则说明没有事务则根据当前方法的事务属性创建connHolder,并使用ThreadLocal保存  
3.connHolder如果存在, 则根据事务方法上的传播特性来决定事务传播行为,创建新事物,或加入当前事务执行,或内嵌事务  
4.在处理事务传播行为时,都要创建新的事务状态对象,然后设置事务同步管理器的事务信息  


## 事务何时提交
执行完被拦截的方法后提交,通过事务管理器来提交,事务管理器会查看事务状态来决定是否提交,是否回滚  
在提交事务时有个判断status.isNewTransaction().必须是新事物才提交,判断条件是存在事务对象和newTransaction =true  
newTransaction字段只有在创建事务事务时设置为true,如果是required传播方法新的事务状态该值为false,内嵌事务该值也是false  
保证事务方法执行完了才能提交,就是在事务入口方法执行完了后才可以提交  

## 事务方法调用另外一个类的非事务方法
事务方法调用当前类的方法,被调用的方法默认被加入到调用方法的事务中执行,不管被调用的方法是否有事务注解  
因为实例方法调用是通过this.methodName(),不是调用代理对象的引用来调用,所以拦截器无法被应用,如果要在实例方法  
中调用另外一个方法并且要执行拦截器,可以使用AopContext.currentProxy(),spring使用ThreadLocal存储代理对象  

如果调用其它类的非事务方法,方法和类上都没有事务注解,如果当前方法处于事务中,那么被调用方法也会加入到当前事务,  
如果代码中进行细粒度的事务控制,只在有数据库操作的地方开启或关闭事务,代码维护时容易出错    
Connection con = DataSourceUtils.getConnection(getDataSource());  
因为被调用的方法是另外一个类中的方法,那么原来的事务方法将会执行完毕,执行doCleanupAfterCompletion(Object transaction)  
如果事务入口的方法执行完毕,会解绑在TransactionSynchronizationManager的conHolder,当非事务方法执行时下面获取连接代码时  
conHolder为空,那么从datasource中获取一个新的数据库连接,如果数据源设置默认不自动提交,jdbc代码中也没有提交事务,那么该事务  
没被提交,jdbc操作也不会有效果,bug就出来了.(有同事为了避免创建事务对象,为了所谓的效率结果导致更多bug),声明式事务就是为了  
避免开发时的事务管理,结果在写代码的过程中还要去分析开启事务后,接下来的逻辑哪里要加事务哪里不加事务注解,无语!!!  

调用其它类,如果使用了事务注解,那么会创建代理,方法调用会执行拦截器,效率是不如不创建代理,但是为了这么一点效率埋下一个雷更不好,  
只有当业务中使用事务注解的地方存在性能瓶颈时再去考虑才是合适的,能正确的运行然后再去考虑效率  
~~~
//事务方法执行doBegin(Object transaction, TransactionDefinition definition)会设置数据库连接,这里可以使用
//
ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
			conHolder.requested();
			if (!conHolder.hasConnection()) {
				logger.debug("Fetching resumed JDBC Connection from DataSource");
				conHolder.setConnection(dataSource.getConnection());
			}
			return conHolder.getConnection();
		}
~~~



AutoProxyCreator的功能是创建代理,所以逻辑只关注是否应对bean创建代理  
// Create proxy if we have advice.  
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);  
创建代理的时候没有先做很多判断,让阅读代码时思路很清晰,在创建代理时根据返回值来标志是否创建代理对象,如果这里包含了  
很多逻辑判断,那么阅读代码时关注点就会被转移,不同的bean是否创建代理的规则不一样,可能规则很复杂,那么阅读代码时思路  
就被打断了,信息量太大,所以以后写代码时要将复杂逻辑用方法包装,让函数入口便得简单,后期维护也很方便  