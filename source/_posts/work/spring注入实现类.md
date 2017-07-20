## spring注入具体实现类
    在使用dubbo时,service中的方法全部对外开放,有些方法不想对外访问,想直接在实现中使用public  
在内部使用时使用实现类注入.
~~~
@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private RoleServiceImpl roleService;


    @Override
    public String getName(Long userId) {
        System.out.println(roleService.getRole(null));
        return "hello";
    }
}
~~~
    上面的代码可能会爆出异常,如果RoleServiceImpl是基于JDK动态代理创建的对象(使用AOP,比如@Transaction注解).  
因为基于接口的JDK动态代理是创建的另一个类并实例化,虽然A1和A2实现相同的接口A,但是A1的引用不能指向A2的实例  
##注入RoleServiceImpl过程
    如果被AOP增强,从spring容器中获取实例时,会被增强.比如使用AbstractAdvisorAutoProxyCreator创建  
代理.  

##解决办法
1.不能使用代理对象,也就是不能使用aop  
2.使用cglib代理,比如使用<tx:annotation-driven proxy-target-class="true"/>  

##思考
    spring'默认按类型注入,在基于接口开发时,如果AutoWired注解的接口有多个实现需要指定@Qualifier  
###spring如何知道一个接口有多个实现?
    猜想是实例化bean的时候,创建一个map,按类型查找实例,
    但是实际是解析依赖时构建一个map.
    DefaultListableBeanFactory.getBeanNamesForType会遍历所有的bean实例,个candicateName  
当有多个候选bean实例时,需要其他信息确定使用哪一个bean实例,这时可以使用@Qualifier注解  

在按类型解析依赖时,DefaultListableBeanFacroty属性Map<Class<?>, String[]> allBeanNamesByType = new ConcurrentHashMap<Class<?>, String[]>(64);缓存了所有类型对应的beanNames, 比如RoleService  有多个实现,那么类型RoleService对应多个bean名字  

    根据接口查找依赖时.遍历所有beanNameDefinitions,根据beanDefinition的name找到对应bean实例  
在判断实例是不是依赖接口的实例,然后统一放入allBeanNamesByType这个map中,所以也就是在解析依赖时  
填充这个map并达到缓存目的,当其他bean依赖需要用到时,直接从缓存中获取就ok了,不用再遍历所有bean定义  

    每个类依赖规则不一样,有的使用@Qualifier注解,有的使用实例名字,所以解析依赖时需要根据bean自身来筛选  
但是allBeanNamesByType是可以重用的,不用再次遍历spring中所有的bean来确定某个接口有哪些实例  

    当一个依赖接口有多个实例对象实现时,需要确定使用哪个,  
1.beanDefinition的primary属性可以确定,在类上使用@Primary注解    
2.依赖的属性名字用实例名字可,例如roleServiceImpl    
3.在依赖属性使用@Qualifier注解,使用该注解解析依赖时有多个实现会选择某一个    


    