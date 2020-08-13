---
layout: post
title: Spring Action 4 读书笔记
category: Java
tags: []
keywords: 
---

#Spring Action 4 读书笔记/n# 装配Bean
##  隐式的bean发现机制和自动装配

    @Component 简单的注解表明该类会作为组件类，并告知Spring要为这个类创建bean
     可以指定bean名，如@Component("guohao")  也可以不写名字 默认为首字母小写的类名
    ==@Named

### 组件扫描
放在配置文件上
#### java类  

    @ComponentScan  放在 @Configuration Spring将会扫描这个包以及这个包下的所有子包，查找带有@Component注解的类
    也可以扫描指定包名@ComponentScan(" ") 或者 @ComponentScan(basePackages=" guohao")
    又或者要扫描多个包，只需要将basePackages属性设置为要扫描包的一个数组即可：@ComponentScan(basePackages={" guohao"，“wenwenwen”})
    又或者 指定所包含的接口和类:@ComponentScan(basePackageClasses={guohao.class，wenwenwen.class})

@ComponentScan(basePackageClasses={guohao.class，wenwenwen.class}
excludeFilters={
    @Filter（.....）
})

过滤到一些东西


#### XML 

    <context:component-scan  base-packge=" "> 填包名 
    Assert.notNull(Object object) 断言某个值 不为空 如果不一样就抛出异常

### 自动装配
@Autowired可以用在构造器上，还可以用在属性的Setter方法上  
@Autowired(required=false) 

>     >  将required属性设置为false时，Spring会尝试执行自动装配，但
>     > 是如果没有匹配的bean的话，Spring将会让这个bean处于未装配的状
>     > 态。但是，把required属性设置为false时，你需要谨慎对待。如 果在你的代码中没有进行null检查的话，这个处于未装配状态的属性
>     > 有可能会出现NullPointerException。
==@Inject

## 通过java代码装配Bean
### 创建配置类
 @Configuration  @Configuration注解表明这个类是一个配置类，该类应该包含在Spring应用上下文中如何创建bean的细节

@Bean注解会告诉Spring这个方法将会返回一个对象，该对象要注册为Spring应用上下文中的bean。方法体中包含了最终产生bean实例的逻辑。
默认情况下，bean的ID与带有@Bean注解的方法名是一样的。 
也可以设置   @Bean("guohao")

### 实现注入
#### 调用方法注入

    @Bean 
    public CDPlayer cdPlayer(){
       return new  CDPlayer(sgtPapper() );
    }

> sgtPeppers()方法上添加了@Bean注解， Spring将会拦截所有对它的调用，并确保直接返回该方法所创建的
> bean，而不是每次都对其进行实际的调用

#### 使用接口 DI方法

     @Bean
    public CDPlayer cdPlayer( CompactDisc compactDisc){
           return new  CDPlayer( compactDisc);
        }
**CompactDisc 是接口** 

> 在这里，cdPlayer()方法请求一个CompactDisc作为参数。当
> Spring调用cdPlayer()创建CDPlayerbean的时候，它会自动装配
> 一个CompactDisc到配置方法之中。然后，方法体就可以按照合适 的方式来使用它。借助这种技术，cdPlayer()方法也能够
> 将CompactDisc注入到CDPlayer的构造器中，而且不用明确引 用CompactDisc的@Bean方法。
> 通过这种方式引用其他的bean通常是最佳的选择，因为它不会要求 将CompactDisc声明到同一个配置类之中。在这里甚至没有要
> 求CompactDisc必须要在JavaConfig中声明，实际上它可以通过组件
> 扫描功能自动发现或者通过XML来进行配置。你可以将配置分散到多 个配置类、XML文件以及自动扫描和装配bean之中，只要功能完整健
> 全即可。不管CompactDisc是采用什么方式创建出来的，Spring都 会将其传入到配置方法中，并用来创建CDPlayer bean。

### Set方法注入
上述是构造器注入  也可以是Set方法注入
![set方法][1]

## 通过XML装配bean
java类只需要一个@Configuration  而 使用XML时，需要在配置文件的顶部声明多个XML模式（XSD）文件
声明 Bean   <bean  class-="com.guohao.helloworld"/> 

> bean的ID将会 是“soundsystem.SgtPeppers#0”。其中，“#0”是一个计数的形
> 式，用来区分相同类型的其他bean。如果你声明了另外一 个SgtPeppers，并且没有明确进行标识，那么它自动得到的ID将会
> 是“soundsystem.SgtPeppers#1”

指定id  
<bean  class-="com.guohao.helloworld" id="guohao"/ >

###  构造器注入初始化bean
#### <constructor-arg   /> 元素

    <bean id=""  class=" "   >
         <constructor-arg    ref ="" />
    </bean>

装配字面量：
构造函数有几个参数就写几个下面这句话
   <constructor-arg    value="" />

装配集合 
 列表为字面量

    <constructor-arg  
            <list
                      <value>  ....</value>
                      <value>  ....</value>
                       <value>  ....</value>
              />
     /> 

在装配集合方面，<constructor-arg>比c-命名空间的属性更有优
势

列表为bean

    <constructor-arg  
                <list
                          <ref   bean="">
                          <ref   bean="">
                          <ref   bean="">
                  />
         /> 
    




#### c名称空间   
 文件头部加一个 xmls:c=" ....................."

    <bean id=""  class=" "    c:cd-ref= " 要注入的beanid" />
    </bean>

装配字面量：
构造函数有几个参数就写几个下面这句话

      c:_0-ref= " 要注入的beanid"
      c:_1-ref= " 要注入的beanid" 

cd为构造器参数名

装配集合


### 属性注入


<property>元素为属性的Setter方法所提供的功能
与<constructor-arg>元素为构造器所提供的功能是一样的

    <bean id="cdplayer" class=" " >
            <property    name=""  ref=" "             / >
    </bean>

    <bean id="cdplayer" class=" " >
                <property    name=""  value=" "             / >
        </bean>

 <bean id="cdplayer" class=" " >
                <property    name=""       

                    <list
                            <vaule>  ..</value> 
                              <vaule> .. </value> 
                            <vaule> ... </value> 
                    />

                / >
        </bean>


p名称空间：

    <bean id="cdplayer" class=" " 
               p:cd-ref="cdplayer"     cd为 set方法参数的变量名
        </bean>

    <bean id="cdplayer" class=" " 
               p:title="guohao"     title 为set方法参数的变量名
                p:artist="wenwen"
        </bean>

### util命名空间  
Spring util-命名空间中的一些功能来简化BlankDiscbean

        
## 导入和混合设置

###  在javaconfig中应用xml配置

    @Import(cdConfig.class)  一个配置类 和另一个配置类相互组合    这个注解是组合java配置类
    
    配置类的整合 最好方式是 ：
    一个最大的配置类  头顶@import({config.class.config2.class})
    
    当在xml中装配bean时 ，
    java类中最大的config头顶应该有 @ImportResource("classpath:cd-config.xml")

### 在xml中 引用javaconfig

    <bean class="guohao.wenConfig"   />
    
    <import resource="cdplayer-config.xml">


## 高级装配
### 配置profile bean

@profile 可以用在类上 ，也可以用在和@bean一起使用  用在配置类上 @Configuation

#### 在xml中配置prioile

用在整个包裹整个xml文件的<beans   profile=“dev” ></beans>
也可以在一个大的xml中  大的

    <beans
        xmlns:......
    xmlns:.........    
    >       
        <beans   profile=“dev” ></beans>
        <beans   profile=“qa” ></beans>
        <beans   profile=“prod” ></beans>
    </beans>

### 激活profile

spring.profiles.default
spring.profiles.active

先找spring.profiles.active 是否设置了  若没有 再找spring.profiles.default，否者就没有可以激活的profile


有多种方式来设置这两个属性：

    作为DispatcherServlet的初始化参数；
    作为Web应用的上下文参数；
    作为JNDI条目；
    作为环境变量；
    作为JVM的系统属性；
    在集成测试类上，使用@ActiveProfiles注解设置。

比如在Web应用的web.xml文件中设置默认的profile


### 条件化bean


@Conditional  它可以用到带有@Bean注解的方法上。如果给定的条件计算结果为true，就会创建这个bean，否则的话，这个bean会被忽略

    @Bean
    @Conditional  (guoaho.class)
    public guoaho guohaoBean{
            ..
    }

guohao这个类必须实现Condition

    public class guohao implements Condition{
           // 复习matches()方法
    
        public boolean matches（ ConditionContext   context，AnnotatedTypeMetadata  metadata）{        
                            Environment env=context.getEnvironment();
                            return env.containsProperty("magic")
                    }
    } 

这个就设计ConditionContext   ，AnnotatedTypeMetadata  这两个类
ConditionContext  :检查bean的定义，是否存在bean，看看环境变量的值，加载的资源，检查类是否存在 等等功能
AnnotatedTypeMetadata  判断带有@Bean注解的方法是不是还有其他特定的注解   借助其他的那些方法，我们能够检查@Bean注解的方法上其他注解的属性


@profile就是  @Conditional实现的

### 处理自动装配的歧义性

@Primary 将与之联用的bean设置为首选。  
可与Component 和 @Bean 连用

#### xml中使用 primary：

    <bean id="wen" class="guohao.class"  primary="true">
    </bean>

@Qualifier   和@Autowired 组合使用 

    @Autowired
    @Qualifier （“iceCream”）
    pub set  （）{
    }

##### 创建自定义限定符

 1. 和Component组合

       @Component
        @Qualifier （“iceCream”）
        pub class Icecream implements Dessert （）{
        .......
        }

 2. 列表项目

            @Bean
            @Qualifier （“iceCream”）
            pub  Dessert Icecream   （）{
                return new IceCream
            }


还可以多个限定符一起用

                @Bean
                @Qualifier （“cold”）
                @Qualifier （“creamy”）
                pub  Dessert Icecream   （）{
                    return new IceCream
                }


但是这种会报错的 java是不能同时添加多个  相同类型的注解
##### 创建自定义限定符

> 它本身 要使用@Qualifier注解来标注。这样我们将不再使
> 用@Qualifier("cold")，而是使用自定义的@Cold注解，该注解 的定义如下所示：

![请输入图片描述][2]

接下来就可以使用 ：

       @Component
        @cold
        pub class Icecream implements Dessert （）{
        .......
        }


### Bean的作用域  
单例 (Singleton)
原型 （多例模式）(Prototype)
会话  (Session)
请求 (Request)

    @Scope   默认是单例的作用域   想要改变时用到
    @Scope（" prototype"） 建议使用 @Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)

可以与@Component  @Bean 组合使用

xml配置如下：

    < bean id="notepad"  class="com.guohao.notepad" scope="prototype"  />


#### 使用会话作用域
@Scope（value=WebApplicationContext.SCOPE_SESSION
                proxyMode=ScopedProxyMode.INTERFACES）

value设置成了WebApplicationContext中的SCOPE_SESSION常量（它的值是session）
ScopedProxyMode.INTERFACES。这个属性解决了将会话或请求作用域的bean注入到单例bean中所遇到的问题

###  运行时注入
平常直接@Bean中注入 值，都是硬编码
为了避免硬编码，可以使用

 1. 属性占位符
 2. SpEL

#### 注入外部的值
使用@PropertySource注解和Environment
JAVA类配置：

    @Configuration
    @PropertySource("classpath: /com/guohao/app.properties")
    pubulic class APPConfig{
            @Autowired
            Environment env ;
                
            @Bean
            public BlankDisc disc(){
                    return new BlankDisc(
                            env.getProperty("disc.title"),
                            env.getProperty("disc.artiist")
                    )
            
        }
        
    }

@PropertySource引用了类路径中一个名为app.properties的文件
中写到：
disc.title  =woshiyigemaobaodexiaohangjia
disc.artist=lalala

getProperty(.....,...)  第二个参数可以时默认值


#### 属性占位符

    <bean  id="sgt"  class="com.guohao" 
            c:_title=${disc.title}"  
            c:_artist=${disc.artist}      
    />

如果我们依赖于组件扫描和自动装配来创建和初始化应用组件的话，
那么就没有指定占位符的配置文件或类了。在这种情况下，我们可以
使用@Value注解，它的使用方式与@Autowired注解非常相似

![请输入图片描述][3]

使用占位符的方法
JAVA类：

> 用PropertySourcesPlaceholderConfigurer，因为它能够基 于Spring
> Environment及其属性源来解析占位符。 如下的@Bean方法在Java中配置了
> PropertySourcesPlaceholderConfigurer
![请输入图片描述][4]

XML配置：
![请输入图片描述][5]

#### 使用Spring表达式语言进行装配


## 面向切面的spring
### 通过切点来选择连接点
#### 编写切点
切点表达式： execution(* concert.Performance.perform(..))
![请输入图片描述][6]
限制匹配
within 

在切点中选择bean

###  通过注解来定义切面
@Aspect
@AspectJ注解进行了标注。该注解表明Audience不仅仅是一个POJO，还是一个切面

    @After    通知方法会在目标方法返回或抛出异常后调用
    @AfterReturning   通知方法会在目标方法返回后调用
    @AfterReturning   通知方法会在目标方法抛出异常后调用
    @Around    通知方法会将目标方法封装起来
    @Before   通知方法会在目标方法调用之前执行


@Pointcut

    @Pointcut注解能够在一个@AspectJ切面内定义可重用的切点
![请输入图片描述][7]

定义好了切面后
想要使用这个切面，必须开启自动代理功能  ，如果光是在配置文件中写一个return bean；
那么只会是 容器中生成一个bean 所以要同时在javaConfig中添加 @ EnableAspectJAutoProxy注解

xml配置：
那么需要使用Springaop命名空间中的<aop:aspectj-autoproxy>元素

### 创建环绕通知
![请输入图片描述][8]

环绕通知 ProceedingJoinPoint 执行proceed方法的作用是让目标方法执行，这也是环绕通知和前置、后置通知方法的一个最大区别。简单理解，环绕通知=前置+目标方法执行+后置通知，proceed方法就是用于启动目标方法执行的

### 处理通知中的参数
![请输入图片描述][9]
然后就可以在通知中加对参数处理的方法

### 通过引入新功能
主要是用@DeclareParents注解

> value属性指定了哪种类型的bean要引入该接口。在本例中，也 就是所有实现Performance的类型。（标记符后面的加号表示
> 是Performance的所有子类型，而不是Performance本 身。）
> defaultImpl属性指定了为引入功能提供实现的类。在这里， 我们指定的是DefaultEncoreable提供实现。
> @DeclareParents注解所标注的静态属性指明了要引入了接 口。在这里，我们所引入的是Encoreable接口。

### 注入Aspect切面  

**这一块  很是看不懂！！！！先暂时放着**
。。

#  构建spring web应用程序
## Spring mvc起步
DispatcherServlet  **前端控制器**

### 跟踪springmvc的请求
### 搭建springmvc

> AbstractAnnotationConfigDispatcherServletInitializer的任意类都会自动地
> 配置Dispatcher-Servlet和Spring应用上下文，Spring的应用上下 文会位于应用程序的Servlet上下文之中
扩展了AbstractAnnotationConfigDispatcherServletInitializer 就相当于同时实现了WebApplicationInitializer
部署到Servlet 3.0容器中的时候，容器会自动发现它，并用它来配置Servlet上下文


#### DispatcherServlet  ContextLoaderListener（Servlet监听器） 两个应用上下文的故事：

在应用启动时，DispatcherServlet   ，他会创建spring的应用上下文。  用于加载配置文件或配置类中所声明的bean，加载包含Web组件的bean，如控制
器、视图解析器以及处理器映射

ContextLoaderListener 会创建另一个应用 上下文要加载应用中的其他bean。这些bean通常是驱动应用后端的中间层和数
据层组件。


AbstractAnnotationConfigDispatcherServletInitializer会同时创建DispatcherServlet和ContextLoaderListener

> GetServletConfigClasses() 方法返回的带有@Configuration注解的类将会用来定
> 义DispatcherServlet应用上下文中的 bean。
> 
> getRootConfigClasses()方法返回的带 有@Configuration注解的类将会用来配
> 置ContextLoaderListener创建的应用上下文中的bean

@EnableWebMvc   启用注解驱动的Spring MVC
xml相对的为：<mvc:annotation-driven>

#### 视图解析器：
![请输入图片描述][10]



 


## 编写基本的控制器
## 接受请求的输入
## 处理表单

# springmvc的高级技术
## springmvc配置的替代方法
### 配置multipart解析器
### 处理multipart请求
## 处理异常
### 将异常映射转为http状态码
为了准确的显示出异常状态，可以手动抛出异常，进行对异常的配置
需要在控制器指定位置抛出这个异常，相应具有404状态码

    @ResponseSratus(value=HttpStatus.Not_Found,
                                    reason="spittle not found "        
                                )
    public class spittlenotfoundException extends RuntimeException{
    }

### 编写异常处理的方法

    @ExceptionHandler (DuplicateSpittleException.class)
    public String handleDuplicateSpittle(){
            return "error/duplicate"
    }

**@ExceptionHandler**注解，当抛出
DuplicateSpittleException异常的时候，将会委托该方法来处
理。
它能处理同一个控制器中所有处理器方法所抛出的异常

## 为控制器添加通知

控制器通知（controller advice）是任意带有@ControllerAdvice注解的类


 1. @ExceptionHandler注解标注的方法；
 2. @InitBinder注解标注的方法；
 3. @ModelAttribute注解标注的方法。

>     带有@ControllerAdvice注解的类中，以上所述的这些方法会 运用到整个应用程序所有控制器中带有@RequestMapping注解的方 法上。

![请输入图片描述][11]

## 跨重定向请求传递数据

什么时候进行重定向：

 1. 不同WEB应用程序之间的访问
 2. 在完成一个post请求过后，进行一次重定向可以避免重新执行post请求的危险


当控制器方法返回的String值
以“redirect:”开头的话，那么这个String不是用来查找视图的，
而是用来指导浏览器进行重定向的路径

    return "redirect:/spitter/"+spitter.getUsername();

### 通过url模板进行重定向

`model.addAttribute("username",spitter.getusername);
model.addAttribute("spitterId",spitter.getid）
return "redirect:/spitter/{username}"`

> 因为模型中的 spitterId属性没有匹配重定向URL中的任何占位符，所以它会自 动以查询参数的形式附加到重定向URL上。

**如果username属性的值是habuma并且spitterId属性的值是42，
那么结果得到的重定向URL路径将会是“/spitter/habuma?spitterId=42”**

### 使用flash属性传递更复杂的值
重定向前：

`model.addFlashAttribute("spitter",spitter)
return "redirect:/spitter/{username}"`

重定向后：
判断传过来的对象是否在模型中：
if(!model.containsAttribute("spitter")){
.....}



# 通过Spring和JDBC征服数据库
## 配置数据源
### 使用JNDI数据源


### 使用数据源连接池
配置DBCPBasicDataSource:
XML配置：
![请输入图片描述][12]
java配置：
![请输入图片描述][13]

### 基于JDBC驱动的数据源
**在Spring中，通过JDBC驱动定义数据源是最简单的配置方式。**

 1. DriverManagerDataSource：在每个连接请求时都会返回一
    个新建的连接。与DBCP的BasicDataSource不同，
    由DriverManagerDataSource提供的连接并没有进行池化管 理；
 2. SimpleDriverDataSource： 与DriverManagerDataSource的工作方式类似，但是它直接
    使用JDBC驱动，来解决在特定环境下的类加载问题，这样的环 境包括OSGi容器；
 3. SingleConnectionDataSource：在每个连接请求时都会返
    回同一个的连接。尽管SingleConnectionDataSource不是 严格意义上的连接池数据源，但是你可以将其视为只有一个连接
    的池

配置DriverManagerDataSource的方法:
java:
![请输入图片描述][14]
xml:
![请输入图片描述][15]

>  与具备池功能的数据源相比，唯一的区别在于这些数据源bean都没有 提供连接池功能，所以没有可配置的池相关的属性。
> 尽管这些数据源对于小应用或开发环境来说是不错的，但是要将其用 于生产环境，你还是需要慎重考虑。因
> 为SingleConnectionDataSource有且只有一个数据库连接，所 以不适合用于多线程的应用程序，最好只在测试的时候使用。
> 而DriverManagerDataSource和SimpleDriverDataSource尽
> 管支持多线程，但是在每次请求连接的时候都会创建新连接，这是以 性能为代价的。鉴于以上的这些限制，我强烈建议应该使用数据源连 接池



### 使用嵌入式数据源

### 使用profile选择数据源
java：
![请输入图片描述][16]
xml：
![请输入图片描述][17]

## 使用jdbc模板

 1. JdbcTemplate：最基本的Spring JDBC模板，这个模板支持简 单的JDBC数据库访问功能以及基于索引参数的查询；
 2. NamedParameterJdbcTemplate：使用该模板类执行查询时 可以将值以命名参数的形式绑定到SQL中，而不是使用简单的索
    引参数；
 3. SimpleJdbcTemplate：该模板类利用Java 5的一些特性如自 动装箱、泛型以及可变参数列表来简化JDBC模板的使用。

使用 jdbcTemplate：

@Bean
public jdbcTemplate jdbcTemplate(DataSource  dataSource ){
            return new jdbcTemplate (datasource);
}

    
@Repository

# 使用springmvc创建rest api
Spring提供了两种方法将资源的Java表述形式转换为发送给客户端的表述形式：

 1. 内容协商（Content negotiation）：选择一个视图，它能够将模型 渲染为呈现给客户端的表述形式；


 2. 消息转换器（Message conversion）：通过一个消息转换器将控 制器所返回的对象转换为呈现给客户端的表述形式。
![请输入图片描述][18]

> @ResponseBody注解会告知Spring，我们要将返回的对象作为资源 发送给客户端，并将其转换为客户端可接受的表述形式。更具体地
> 讲，DispatcherServlet将会考虑到请求中Accept头部信息，并 查找能够为客户端提供所需表述形式的消息转换器
> 
> 举例来讲，假设客户端的Accept头部信息表明它接 受“application/json”，并且Jackson JSON库位于应用的类路径
> 下，那么将会选择MappingJacksonHttpMessage-Converter
> 或MappingJackson2HttpMessageConverter（这取决于类路径
> 下是哪个版本的Jackson）。消息转换器会将控制器返回的Spittle列表 转换为JSON文档，并将其写入到响应体中

`@RequestMapping注解。在这里，我使用了produces属性表明这
个方法只处理预期输出为JSON的请求。也就是说，这个方法只会处
理Accept头部信息包含“application/json”的请求。其他任何类
型的请求，即使它的URL匹配指定的路径并且是GET请求也不会被这
个方法处理。这样的请求会被其他的方法来进行处理（如果存在适当
方法的话），或者返回客户端HTTP 406（Not Acceptable）响应`

@RequestMapping注解。在这里，我使用了produces属性表明这
个方法只处理预期输出为JSON的请求。

@RequestBody也能告诉Spring查找一个消息转换器，将来自客户端的资源表述转换为对象。例如，假设
我们需要一种方式将客户端提交的新Spittle保存起来
一般前台会传过来一个json数据  ，把json数据转换为指定对象的pojo
把前台传过来的数据转换为pojo



![请输入图片描述][19]

consumes 属性：
传进来mvc中的格式


> consumes属性的工作方式类似于 produces，不过它会关注请求的Content-Type头部信息。它会告
> 诉Spring这个方法只会处理对“/spittles”的POST请求，并且要求
> 请求的Content-Type头部信息为“application/json”

produces属性：
传到客户端的格式







## 发送错误信息到客户端

 1. 使用@ResponseStatus注解可以指定状态码；
 2. 控制器方法可以返回ResponseEntity对象，该对象能够包含 更多响应相关的元数据；
 3. 异常处理器能够应对错误场景，这样处理器方法就能关注于正常 的状况。

### 使用ResponseEntity

> ResponseEntity：@ResponseBody的替代方案，控制器方法可以返回一
> 个ResponseEntity对象。ResponseEntity中可以包含响应相关
> 的元数据（如头部信息和状态码）以及要转换成资源表述的对象。

![请输入图片描述][20]

最后，会创建一个新的ResponseEntity，它会把Spittle和状态码传送给客户端。

为了更好的反射错误 可以定义一个error类
然后把error装进ResponseEntity<object>


# 使用对象-关系映射持久化数据
## 在spring中集成Hibernate
### 声明Hibernate的session工厂
### 构建不依赖于spring的Hibernate
## spring与java持久化API
### 配置实体管理器工厂

### 编写基于JPA的Repository
JPA定义了两种类型的实体管理器：
**通过EntityManagerFactory获取EntityManager实例**

 1. 应用程序管理类型（Application-managed）  应用程序管理类型的EntityManager是由EntityManagerFactory创建的
 2. 容器管理类型（Container-managed）     而后者是通过PersistenceProvider的createEntityManagerFactory()方法得到的。

>     不管你希望使用哪种EntityManagerFactory，Spring
>     都会负责管理EntityManager。如果你使用的是应用程序管理类型
>     的实体管理器，Spring承担了应用程序的角色并以透明的方式处
>     理EntityManager。在容器管理的场景下，Spring会担当容器的角
>     色。

这两种实体管理器工厂分别由对应的Spring工厂Bean创建：

 1. LocalEntityManagerFactoryBean生成应用程序管理类型 的EntityManager-Factory；
 2. LocalContainerEntityManagerFactoryBean生成容器管理类型的Entity-ManagerFactory。

#### 配置应用程序管理类型的JPA
它绝大部分配置信息来源于一个名为persistence.xml的配置文件。这个文件必须位于类路
径下的META-INF目录下。


persistence.xml的作用在于定义一个或多个持久化单元
![请输入图片描述][21]
![请输入图片描述][22]



#### 配置 容器管理的JPA
将数据源信息配置在Spring应用上下文中：
![请输入图片描述][23]

jpaVendorAdapter属性用于指明所使用的是哪一个厂商的JPA实现。Spring提供了多个JPA厂商适配器：

 1. EclipseLinkJpaVendorAdapter
 2. HibernateJpaVendorAdapter
 3. OpenJpaVendorAdapter

Hibernate作为JPA实现：

> persistence.xml文件的主要作用就在于识别持久化单元中的实体类。 但是从Spring 3.1开始，我们能够
> 在LocalContainerEntityManagerFactoryBean中直接设 置packagesToScan属性：

@Entity  ：

    LocalContainerEntityManagerFactoryBean
    会扫描com.habuma.spittr.domain包，查找带有@Entity注解
    的类。

#### 从JNDI获取实体管理器工厂
。。
###借助SpringData实现自动化的JPA  Repository
1 以接口定义的方式创建Repositoyr
public interface SpitterRepository extend JpaRepository<spitter,Long>

> 这里，SpitterRepository扩展了Spring Data JPA的 JpaRepository

同时你不需要去实现他的持久化方法，spring data jpa会为我们自动做好这件事情
 
但是要配置一下类似于组件扫描的:
xml方式：
<jpa:repositories base-package="com.guohao" />

> <jpa:repositories>会扫描它的基础包来查找扩展自Spring Data JPA
> Repository接口的所有接口。如果发现了扩展自 Repository的接口，它会自动生成（在应用启动的时候）这个接口 的实现。


java配置：
配置类上添加@EnableJpaRepositories

#### 定义查询方法

        public interface SpitterRepository extend JpaRepository<spitter,Long>{
                Spitter findByUsername （String  username）
    }

Repository方法的命名遵循一种模式:
![请输入图片描述][24]

对于大部分情况主题会被省略:
要查询的对象类型是通过如何参数化JpaRepository接口来确定的，而不是方法名称中的主题

省略一些部分：
....
@Query：

    @Query("select s from spitter s where s.email like 'gmail.com' ")
       List<spitter>  findAllGmailSpitters();






  [1]: http://139.199.206.151:1998/upload/2018/12/vlf24aq3peha9ptslpuujma63t.bmp
  [2]: http://139.199.206.151:1998/upload/2018/12/seief0ni4agdips46csfspcpcn.bmp
  [3]: http://139.199.206.151:1998/upload/2018/12/rrj1l6kpbmgpqo2ur75jljmgch.bmp
  [4]: http://139.199.206.151:1998/upload/2018/12/20uf09nt3ig0no9d8pqbm4i85e.bmp
  [5]: http://139.199.206.151:1998/upload/2018/12/u3htsknibihsqqn6gt4rvbs3rl.bmp
  [6]: http://139.199.206.151:1998/upload/2018/12/vu0al3fd08g29rugu7o11lqivu.bmp
  [7]: http://139.199.206.151:1998/upload/2018/12/qqqgefcrdcjpgohpljpaqf273a.png
  [8]: http://139.199.206.151:1998/upload/2018/12/rlfkrijbneiekoo554lhgh90qe.png
  [9]: http://139.199.206.151:1998/upload/2018/12/7rfvi2ren4h70qrs1ed8gnt0gp.png
  [10]: http://139.199.206.151:1998/upload/2018/12/7up18kvpb6ie7rjbe0m8125fsv.jpg
  [11]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/5gmmp1aieugkqpcteb0vf94rbg.jpg
  [12]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/7fdl2e4vdij0jro2f3f17alls3.jpg
  [13]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/7gk4qu8ld6httolmo0v8v0qh54.png
  [14]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/5rlaom6u0kgvrpud755kv1m2qv.bmp
  [15]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/46at7itsjkj91oadco4aout2r2.bmp
  [16]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/o4prl5ku0ih5trsmf0itrprb7q.png
  [17]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/u8jk2a049sjscphsqie9gvp00r.png
  [18]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/s6gj6a72iigahq5sm3a0gs6uuk.bmp
  [19]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/318kvqufc8hilqiv0soe5qn8ee.bmp
  [20]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/v90265cfmaivtpeipuni1pu6bp.bmp
  [21]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/oc109q7gg2i9bpjvbunh1dbaji.bmp
  [22]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/rsrgk6j2r4hr2oikfpbraba87v.bmp
  [23]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/5fohg21ieogrhogv45tqnl98tf.bmp
  [24]: http://www.hjslihaoaijiaqi.club:1998/upload/2018/12/7fhguhu5figd3ohqm5n5q5jfrv.bmphttp://www.hjslihaoaijiaqi.club:1998/upload/2018/12/7fhguhu5figd3ohqm5n5q5jfrv.bmp