# 								Spring源码分析

##                              一.Spring启动过程分析

### 1.Spring简介

spring的最基本的功能就是创建对象及管理这些对象之间的依赖关系，实现低耦合、高内聚。还提供像通用日志记录、性能统计、安全控制、异常处理等面向切面的能力，还能帮我们管理最头疼的数据库事务，本身提供了一套简单的JDBC访问实现，提供与 第三方数据访问框架集成（如Hibernate、JPA），与各种Java EE技术整合（如Java Mail、任务调度等等），提供一套自己的web层框架Spring MVC、而且还能非常简单的与第三方web框架集成。从这里我们可以认为Spring是一个超级粘合平台，除了自己提供功能外，还提供粘合其他技术和框架 的能力，从而使我们可以更自由的选择到底使用什么技术进行开发。而且不管是JAVA SE（C/S架构）应用程序还是JAVA EE（B/S架构）应用程序都可以使用这个平台进行开发。

### 2.Spring的运行流程

spring的启动过程其实就是其IoC容器的启动过程，对于web程序，IoC容器启动过程即是建立上下文的过程。

web.xml配置如下：     

```xml
<context-param>
	<param-name>contextConfigLocation</param-name>
	<param-value>classpath:applicationcontext.xml</param-value>
</context-param>

<listener>
	<listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

我们从xml配置文件分析：

<context-param></context-param> 这对标签是去加载Spring的核心配置文件

<listener></listener>这对标签是监听Spring容器的启动，ContextLoaderListener是实现了ServletContextListener接口的监听类，监听到项目启动时会触发contextInitialized方法（该方法主要完成ApplicationContext对象的创建），关闭时会触发contextDestroyed方法（清理ApplicationContext）

①项目启动触发contextInitialized方法

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener
```

该方法就只做一件事：通过父类ContextLoader的initWebApplicationContext创建Spring的上下文对象

②initWebApplicationContext方法做了三件事：

创建WebApplicationContext；

加载Spring文件里的Bean实例；

将WebApplicationContext放入ServletContext（JavaWeb全局变量）中；

③createWebApplicationContext创建上下文对象，支持用户自定义的上下文对象，但必须继承ConfigurableWebApplicationContext

④configureAndRefreshWebApplicationContext方法用于封装ApplicationContext数据并且初始化所有相关的Bean对象。它从web.xml中读取名为contextConfigLocation的配置，这就是Spring xml的数据源设置，然后放到ApplicationContext中，最后就是refresh方法执行所有Java对象的创建。

⑤完成ApplicationContext创建之后就是将其放入ServletContext中。

```java
<servlet>
	<servlet-name>DispatcherServlet</servlet-name>
	<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
	<load-on-startup>1</load-on-startup>
</servlet>

<!-- Map all requests to the DispatcherServlet for handling -->
<servlet-mapping>
	<servlet-name>DispatcherServlet</servlet-name>
	<url-pattern>/</url-pattern>
</servlet-mapping>
```

这段代码是初始化DispatcherServlet，<load-on-startup>是设置启动容器就初始化改Servlet

<url-pattern>是表示哪些请求交给SpringMVC处理

### 3.总结Spring的启动过程

1.首先，对于一个web应用，其部署在web容器中，web容器提供其一个全局的上下文环境，这个上下文就是ServletContext，其为后面的spring IoC容器提供宿主环境；

2.其 次，在web.xml中会提供有contextLoaderListener。在web容器启动时，会触发容器初始化事件，此时 contextLoaderListener会监听到这个事件，其contextInitialized方法会被调用，在这个方法中，spring会初始 化一个启动上下文，这个上下文被称为根上下文，即WebApplicationContext，这是一个接口类，确切的说，其实际的实现类是 XmlWebApplicationContext。这个就是spring的IoC容器，其对应的Bean定义的配置由web.xml中的 context-param标签指定。在这个IoC容器初始化完毕后，spring以WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE为属性Key，将其存储到ServletContext中，便于获取；

3.再次，contextLoaderListener监听器初始化完毕后，开始初始化web.xml中配置的Servlet，这里是DispatcherServlet，这个servlet实际上是一个标准的前端控制器，用以转发、匹配、处理每个servlet请 求。DispatcherServlet上下文在初始化的时候会建立自己的IoC上下文，用以持有spring mvc相关的bean。在建立DispatcherServlet自己的IoC上下文时，会利用WebApplicationContext.ROOTWEBAPPLICATIONCONTEXTATTRIBUTE 先从ServletContext中获取之前的根上下文(即WebApplicationContext)作为自己上下文的parent上下文。有了这个 parent上下文之后，再初始化自己持有的上下文。这个DispatcherServlet初始化自己上下文的工作在其initStrategies方 法中可以看到，大概的工作就是初始化处理器映射、视图解析等。这个servlet自己持有的上下文默认实现类也是 mlWebApplicationContext。初始化完毕后，spring以与servlet的名字相关(此处不是简单的以servlet名为 Key，而是通过一些转换，具体可自行查看源码)的属性为属性Key，也将其存到ServletContext中，以便后续使用。这样每个servlet 就持有自己的上下文，即拥有自己独立的bean空间，同时各个servlet共享相同的bean，即根上下文(第2步中初始化的上下文)定义的那些 bean。

## 二.Spring：源码解读Spring IOC原理

### 1.什么是Ioc(**inversion of Contro**l)/DI(**dependency injection**)？

**IoC 容器**：最主要是完成了完成对象的创建和依赖的管理注入等等。

先从我们自己设计这样一个视角来考虑：

所谓控制反转，就是把原先我们代码里面需要实现的对象创建、依赖的代码，反转给容器来帮忙实现。那么必然的我们需要创建一个容器，同时需要一种描述来让容器知道需要创建的对象与对象的关系。这个描述最具体表现就是我们可配置的文件。

一：实例化对象有时候并不容易，有时候对象与对象之间关系很复杂，我们去维护十分复杂。

二：我们都交给容器，实现解藕

三：我们在类的生产过程中，要对类做一些事情，比如懒加载，代理，我们都交给容器，我们就不需要关心代理的过程是什么样的。

**DI**：为了实现Ioc这个目的，我们可以使用DI（依赖注入）

依赖：在A类中有一个B类的属性，我们就认为A类依赖B类。

注入：注入就是把需要的属性填充到依赖的类中，Spring中大致的过程就是容器初始化B类，然后用一些办法把属性填充到A类中，这里面的一些办法，有两种，构造方法和setter注入。

容器：Spring就是map，就是一个能够容纳对象的器件。

### 2.Spring IOC体系结构？

(1) BeanFactory

​         Spring Bean的创建是典型的工厂模式，这一系列的Bean工厂，也即IOC容器为开发者管理对象间的依赖关系提供了很多便利和基础服务，在Spring中有许多的IOC容器的实现供用户选择和使用，其相互关系如下：

![Spring启动过程](/Library/Typora所需图/Spring启动过程.png)

其中BeanFactory作为最顶层的一个接口类，它定义了IOC容器的基本功能规范，BeanFactory 有三个子类：ListableBeanFactory、HierarchicalBeanFactory 和AutowireCapableBeanFactory。但是从上图中我们可以发现最终的默认实现类是 DefaultListableBeanFactory，他实现了所有的接口。那为何要定义这么多层次的接口呢？查阅这些接口的源码和说明发现，每个接口都有他使用的场合，它主要是为了区分在 Spring 内部在操作过程中对象的传递和转化过程中，对对象的数据访问所做的限制。例如 ListableBeanFactory 接口表示这些 Bean 是可列表的，而 HierarchicalBeanFactory 表示的是这些 Bean 是有继承关系的，也就是每个Bean 有可能有父 Bean。AutowireCapableBeanFactory 接口定义 Bean 的自动装配规则。这四个接口共同定义了 Bean 的集合、Bean 之间的关系、以及 Bean 行为.

最基本的IOC容器接口BeanFactory，我们假想工厂就是IOC容器：

```java
1 public interface BeanFactory {    
2      
3      //对FactoryBean的转义定义，因为如果使用bean的名字检索FactoryBean得到的对象是工厂生成的对象， 
4      //如果需要得到工厂本身，需要转义           
5      String FACTORY_BEAN_PREFIX = "&"; 
6         
7      //根据bean的名字，获取在IOC容器中得到bean实例    
8      Object getBean(String name) throws BeansException;    
9    
10     //根据bean的名字和Class类型来得到bean实例，增加了类型安全验证机制。    
11      Object getBean(String name, Class requiredType) throws BeansException;    
12     
13     //提供对bean的检索，看看是否在IOC容器有这个名字的bean    
14      boolean containsBean(String name);    
15     
16     //根据bean名字得到bean实例，并同时判断这个bean是不是单例    
17     boolean isSingleton(String name) throws NoSuchBeanDefinitionException;    
18     
19     //得到bean实例的Class类型    
20     Class getType(String name) throws NoSuchBeanDefinitionException;    
21     
22     //得到bean的别名，如果根据别名检索，那么其原名也会被检索出来    
23    String[] getAliases(String name);    
24 }
```

在BeanFactory里只对IOC容器的基本行为作了定义，根本不关心你的bean是如何定义怎样加载的。正如我们只关心工厂里得到什么的产品对象，至于工厂是怎么生产这些对象的，这个基本的接口不关心。

 		而要知道工厂是如何产生对象的，我们需要看具体的IOC容器实现，spring提供了许多IOC容器的实现类。比如XmlBeanFactory，ClasspathXmlApplicationContext等。其中XmlBeanFactory就是针对最基本的ioc容器的实现，这个IOC容器可以读取XML文件定义的BeanDefinition（XML文件中对bean的描述）,如果说XmlBeanFactory是容器中的屌丝，**<u>ApplicationContext</u>**应该算容器中的高帅富.

​		**ApplicationContext**是Spring提供的一个高级的IoC容器，它除了能够提供IoC容器的基本功能外，还为用户提供了以下的附加服务。

从**<u>ApplicationContext</u>**接口的实现，我们看出其特点： 

​         1.  支持信息源，可以实现国际化。（实现MessageSource接口）

​         2.  访问资源。(实现ResourcePatternResolver接口，这个后面要讲)

​         3.  支持应用事件。(实现ApplicationEventPublisher接口)

### 3.IoC容器的初始化？

​		IOC容器的初始化包括BeanDinfinition的Resource定位，载入和注册这三个基本过程。我们以ApplicationContext为例，ApplicationContext系列容器是最熟悉的，因为web项目中使用的XmlWebApplicationContext，还有ClasspathXmlApplicationContext等

![SpringIOC初始化](/Library/Typora所需图/SpringIOC初始化.png)

## 三.Spring的循环依赖

### 1.springbean的初始化过程

spring在单例情况下是默认支持循环依赖的，下面我们来证明：

循环依赖：就是A类中有B类的引用，B类中有A类的引用，那么这里就要说到Spring的一个功能：依赖注入。

依赖注入是在Spring初始化的时候完成的，初始化一个bean，bean有一个初始化的过程，这个过程就是Spring bean的生命周期。

Springbean是从哪里产生的呢：先产生一个类

Class---------BeanDefinition--------Object(Bean)

**BeanDefinition：**

BeanDefinition是一个接口，它有很多实现类，在你添加了一些Spring的组建后，比如@compontent，@Lazy等，这些都需要在这个类被加载后，这些信息要先放入BeanDefinition对象中，然后多个类加载会对应多个对象，这些对象会被存入一个map中，后面会有一个函数遍历map，以便于后面实例化这个类时用到

讲循环依赖之前，我们先看一看一个对象走完Spring的生命周期，变成一个spring bean的过程是怎样的，换句话说，就是Spring是怎么初始化一个类对象，然后将属性注入的。这里是Spring很庞大的一个过程，我们就只关心初始化有关的变量和函数，很多函数和代码我们可以忽略。

```java
/*要初始化的Student类源码*/
@Component
public class Student {

    @Autowired
    teacher t1;
  
    public Student() {
        System.out.println("Construct Student is successful!");
    }

}
```

```java
/*要初始化的teacher类源码*/
@Component
public class teacher {
    public teacher() {
        System.out.println("Construct teacher is successful!");
    }
}
```

```java
/*配置类，配置你要扫描的包*/
@ComponentScan("com.Spring")
public class SpringConfig {}
```

```java
/*测试类的初始化Spring容器的代码*/
public class test {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext = new AnnotationConfigApplicationContext(SpringConfig.class);
        Student bean = applicationContext.getBean(Student.class);
        System.out.println(bean);
    }
}
```

***1.refresh()中完成对Bean的初始化，所以我们直接看refresh()方法***

```java
public AnnotationConfigApplicationContext(Class<?>... annotatedClasses) {
    this();
    this.register(annotatedClasses);//this.beanDefinitionMap.put(beanName, beanDefinition)
    this.refresh();
}
```

****

***2.refresh方法中调用了很多方法，我们第一个要看的其实是invokeBeanFactoryPostProcessors这个方法，这个方法是完成对bean的扫描，这里只是列出了部分方法***

扫描前：

![](/Library/Typora所需图/扫描Bean的信息.png)

```java
public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
              
              	//完成所谓的扫描，
                this.invokeBeanFactoryPostProcessors(beanFactory);
              	
              	//在这个类中完成对象的初始化
              	this.finishBeanFactoryInitialization(beanFactory);
            }
}
```

扫描后：我们可以发现在这个map中多了一个我们自己的类的信息，注意这时候还没有初始化，beanDefinitionMap是存放类信息的map这时候只是扫描出了有我们自己定义的类。就是添加了@Component组件的spring都会扫描到。

![](/Library/Typora所需图/扫描完成bean的信息.png)

这个map的key是类名小写的字符串，value里就是扫描出来的类的信息，问题来了，为什么多了扫描这一步，我们都知道JVM在类加载的时候，会加载类的名字等，判断 构造函数等等，为什么还需要spring自己扫描呢，因为 在spring当中还多了是否单例，是都懒加载等等一系列信息，所以需要扫描出来判断，我们点开看看（只截取了 部分，有一个scope值是单例的）。

![](/Library/Typora所需图/beanDefination的value.png)

***

***3.接着我们看this.finishBeanFactoryInitialization(beanFactory)这个方法，这个方法之后的 一系列方法 都是为完成这个对象的初始化做准备，会初始化我们的对象其中最重要的就是beanFactory.preInstantiateSingletons()这个方法***

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
  /*前面部分代码 略*/
  beanFactory.preInstantiateSingletons();
}
```

***

***4.在preInstantiateSingletons()方法中，有一个List，它的作用是用beanDefinitionNames去map中遍历我们所有的bean，我们可以看到它进行了许多do...while...验证，那个存放beanDefinition的map名字是BeanFactoryPostProcessor，就是判断是否单例等，验证通过之后，然后最后调用getBean***

```java
 public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Pre-instantiating singletons in " + this);
        }

        List<String> beanNames = new ArrayList(this.beanDefinitionNames);
        Iterator var2 = beanNames.iterator();

        while(true) {
            String beanName;
            Object bean;
            do {
                while(true) {
                    RootBeanDefinition bd;
                    do {
                        do {
                            do {
                                if (!var2.hasNext()) {
                                    var2 = beanNames.iterator();
                            				/*省略部分代码*/
                            } while(bd.isAbstract());
                        } while(!bd.isSingleton());
                    } while(bd.isLazyInit());

                    if (this.isFactoryBean(beanName)) {
                        bean = this.getBean("&" + beanName);
                        break;
                    }

                    this.getBean(beanName);
                }
            } while(!(bean instanceof FactoryBean));

            FactoryBean<?> factory = (FactoryBean)bean;
            boolean isEagerInit;
            if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                SmartFactoryBean var10000 = (SmartFactoryBean)factory;
                ((SmartFactoryBean)factory).getClass();
                isEagerInit = (Boolean)AccessController.doPrivileged(var10000::isEagerInit, this.getAccessControlContext());
            } else {
                isEagerInit = factory instanceof SmartFactoryBean && ((SmartFactoryBean)factory).isEagerInit();
            }

            if (isEagerInit) {
                this.getBean(beanName);
            }
        }
    }
```

****

***5.getBean又调用了doGetBean,所以我们只看doGetBean，这是部分代码，在这部分代码中有一个非常重要的方法就是getSingleton(beanName)，这个方法在doGetBean中调用了两次，下面我们画大篇幅讲解一下这个方法，在第二次调用这个方法会调用createBean(beanName, mbd, args)方法，在这个createBean方法之前的所有动作，我们都可以说它是验证，验证有没有方法依赖（LookUp），是否单例等 ，在creatBean中才开始真正的创建bean***

```java
protected <T> T doGetBean(String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly) throws BeansException {
 String beanName = this.transformedBeanName(name);
  			/*第一次调用getSingleton*/
        Object sharedInstance = this.getSingleton(beanName);
        Object bean;
        if (sharedInstance != null && args == null) {
            /*省略部分代码*/
        } else {
            if (this.isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            BeanFactory parentBeanFactory = this.getParentBeanFactory();
            if (parentBeanFactory != null && !this.containsBeanDefinition(beanName)) {
                String nameToLookup = this.originalBeanName(name);
                if (parentBeanFactory instanceof AbstractBeanFactory) {
                    return ((AbstractBeanFactory)parentBeanFactory).doGetBean(nameToLookup, requiredType, args, typeCheckOnly);
                }

                if (args != null) {
                    return parentBeanFactory.getBean(nameToLookup, args);
                }

                if (requiredType != null) {
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }

                return parentBeanFactory.getBean(nameToLookup);
            }

  /*省略部分代码*/
  if (mbd.isSingleton()) {
    /*第二次调用getSingleton*/
    sharedInstance = this.getSingleton(beanName, () -> {
      try {
        return this.createBean(beanName, mbd, args);
      } catch (BeansException var5) {
        this.destroySingleton(beanName);
        throw var5;
      }
    });
    bean = this.getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
  }
  /*省略部分代码*/
}
```

**getSingleton**：在**getSingleton**方法中又调用了**getSingleton**，我们点到这个**getSingleton**中。

**注意！！！！！！这里是第一次调用getSingleton**

我们看下图，**Object singletonObject = this.singletonObjects.get(beanName)**，这就意味着这个方法第一次被调用的时候就会去**singletonObjects**这个map里面去拿bean，这个map叫做spring的单例池，存放已经初始化好的对象，你可以把它理解为一个缓存，那么我们可以想到，在一个对象第一次走spring的生命周期，它去这个map中拿，一定是拿不到的，所以第一次返回singletonObject一定等于null，于是又返回到doGetBean方法

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            synchronized(this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }

        return singletonObject;
    }
```

接着，会执行doGetBean方法中的第二次**getSingleton**。

**注意！！！！！！这里是第二次调用是方法的重载！！！！**

我们看看第二次调用都干了什么：大家只需要记住这里面主要是将这个类加入一个set集合意为正在创建的bean，就是说还没有创建完成。

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized(this.singletonObjects) {
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                if (this.singletonsCurrentlyInDestruction) {
                    throw new BeanCreationNotAllowedException(beanName, "Singleton bean creation not allowed while singletons of this factory are in destruction (Do not request a bean from a BeanFactory in a destroy method implementation!)");
                }

                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
                }
								//这里这个方法是将正在创建的这个对象加入一个set集合，这个集合装的是正在创建的对象
                this.beforeSingletonCreation(beanName);
                boolean newSingleton = false;
                boolean recordSuppressedExceptions = this.suppressedExceptions == null;
    						/*省略部分代码*/
    }
```

****

***6.createBean中，它会先拿BeanDefinition对象，然后从中取出对象的类信息，然后后面会调用doCreateBean我们重点看doCreateBean***

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        if (this.logger.isTraceEnabled()) {
            this.logger.trace("Creating instance of bean '" + beanName + "'");
        }

        RootBeanDefinition mbdToUse = mbd;
        Class<?> resolvedClass = this.resolveBeanClass(mbd, beanName, new Class[0]);
        if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
            mbdToUse = new RootBeanDefinition(mbd);
            mbdToUse.setBeanClass(resolvedClass);
        }

        try {
            mbdToUse.prepareMethodOverrides();
        } catch (BeanDefinitionValidationException var9) {
            throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(), beanName, "Validation of method overrides failed", var9);
        }

        Object beanInstance;
        try {
            //这里第一次调用后置处理器 
            beanInstance = this.resolveBeforeInstantiation(beanName, mbdToUse);
            if (beanInstance != null) {
                return beanInstance;
            }
        } catch (Throwable var10) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "BeanPostProcessor before instantiation of bean failed", var10);
        }

        try {
            beanInstance = this.doCreateBean(beanName, mbdToUse, args);
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Finished creating instance of bean '" + beanName + "'");
            }

            return beanInstance;
        } catch (ImplicitlyAppearedSingletonException | BeanCreationException var7) {
            throw var7;
        } catch (Throwable var8) {
            throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", var8);
        }
    }
```

***

***7.doCreateBean方法，主要是在调用this.createBeanInstance(beanName, mbd, args)方法后，可以看看控制台，就打印了构造函数的内容，然后在第四次调用后置处理的时候，判断是否需要AOP，这里将一个工厂对象加入了SingletonFactories的map，这是我提到的第二个map到这里（很重要）。还没有结束，按照我们以往创建对象的逻辑，new一个对象时自动调用构造方法，并且类里面的属性都是自动填充好了，但是在这里，spring走到这一步仅仅完成了对一个对象的创建，也就是调用了构造方法，但是，里面的属性，一个都没有注入。***

```java

protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);
        }

        if (instanceWrapper == null) {
          //第二次调用后置处理器，实例化对象，这里其实就是推断出一个合理的构造方法调用它
            instanceWrapper = this.createBeanInstance(beanName, mbd, args);
        }
				
        Object bean = instanceWrapper.getWrappedInstance();
        Class<?> beanType = instanceWrapper.getWrappedClass();
        if (beanType != NullBean.class) {
            mbd.resolvedTargetType = beanType;
        }

        synchronized(mbd.postProcessingLock) {
            if (!mbd.postProcessed) {
                try {
                  	//第三次调用后置处理器
                    this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
                } catch (Throwable var17) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);
                }

                mbd.postProcessed = true;
              
            }
        }
  
				//判断是否需要循环依赖，默认是支持循环依赖的，因为allowCircularReferences这个参数spring默认为true
        boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);
        if (earlySingletonExposure) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");
            }
						//第四次调用后置处理器，判断是否需要AOP，这里将一个工厂对象加入了SingletonFactories的map，这个map的，这是我提到的第二个map
            this.addSingletonFactory(beanName, () -> {
                return this.getEarlyBeanReference(beanName, mbd, bean);
            });
          
        }

        Object exposedObject = bean;

        try {
          	//填充属性，也就是自动注入，这里会进行第五次和第六次后置处理器调用
            this.populateBean(beanName, mbd, instanceWrapper);
          	//里面会进行第八次第九次后置处理器的调用，这里面会进行初始化方法的调用
            exposedObject = this.initializeBean(beanName, exposedObject, mbd);
        } catch (Throwable var18) {
            if (var18 instanceof BeanCreationException && beanName.equals(((BeanCreationException)var18).getBeanName())) {
                throw (BeanCreationException)var18;
            }
          /*省略部分代码*/
```

****

***8.填充属性（自动注入）在populateBean(beanName, mbd, instanceWrapper)中完成，这个方法的代码量十分庞大我只截了一小部分，这个循环是完成属性注入的地方，为什么是循环呢，spring是把所有的要用到的bean都拿了出来， InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp，在这行代码中，ibp就是我们通过debug可以看清楚此时spring循环到什么后置处理器了(其实提了这么多后置处理器，后置处理器其实就是来改变这些个bean的方法，专业一点就叫做后置处理器)，我们执行一遍后发现，ibp={CommonAnnotaionBeanPostProcessor@xxxx}，请大家记住这个后置类CommonAnnotaionBeanPostProcessor的名字，这个类当你在代码中用到@Resource时调用，但是我的代码中用到的是@Autowired，所以它还是没有完成属性的注入，那么什么@Autowired用什么类呢，ibq={AutowiredAnnotaionBeanPostProcessor@xxxx}，这个类专门处理@Autowired***

```java
while(var9.hasNext()) {
  BeanPostProcessor bp = (BeanPostProcessor)var9.next();
  if (bp instanceof InstantiationAwareBeanPostProcessor) {
    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor)bp;
    PropertyValues pvsToUse = ibp.postProcessProperties((PropertyValues)pvs, 		       bw.getWrappedInstance(), beanName);
    if (pvsToUse == null) {
      if (filteredPds == null) {
        filteredPds = this.filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
      }

      pvsToUse = ibp.postProcessPropertyValues((PropertyValues)pvs, filteredPds, bw.getWrappedInstance(), beanName);
      if (pvsToUse == null) {
        return;
      }
    }

    pvs = pvsToUse;
  }
}
```

***

好了，到这里是我们单独一个bean的生命周期走完了，也就是到这一步，构造方法被调用了，并且属性也会被填充（如果没有循环依赖的话），这是很简单的一个步骤，下面我们加上循环依赖

我们将teacher类的代码改为如下：

```java
@Component
public class teacher {

    @Autowired
    private Student s1;

    public teacher() {
        System.out.println("Construct teacher is successful!");
    }
}
```

### 2.循环依赖的解决

当我们在teacher类中添加Student类的引用，这时候就会产生循环依赖。我们来看看spring是怎么解决的。

就像上面所说的前面都不会变，直到doCreatBean方法中注入属性的时候！！！！！！一切都变了。

我们可以通过debug方式发现，在注入属性并且后置处理器变为处理Autowired注解的时候，代码会跳转到第一次调用getSingleton方法那里，为什么呢？因为我们需要teacher对象，但是，teacher从没有在spring中出现过，所以 spring会去先拿到 teacher对象，这时候的beanName已经变为了teacher，换句话说，当spring发现你需要teacher类的对象时，然后就会把刚刚创建Student的方法走一遍，这时候要注意，但是有些地方会不一样。

不一样的地方在于，teacher对象在第二次调用getSingleton方法的时候 ，我们前面提到过这个方法会将正在创建的类加入一个set集合，这时候set集合中就会出现两个元素：Student和teacher，然后向下执行这时候又和前面一样，去判断构造函数创建这个对象，然后调用第四次后置处理器，将这个对象加入二级缓存就是进工厂对它加工，然后进行属性注入，在注入的时候一样，循环到Autowired的注解时，发现，又需要Student的对象，这时候！！！！！！又回到了getSingleton()，这时候我再来放上getSingleton的源码，**注意！！！这里是调用第一个getSingleton方法**：

别看只有这么几行，但是这才是精华，我们看到还是从spring的单例池（singletonObjects这个map）中去拿已经初始化好的bean，好的依旧拿不到，因为这时候两个bean还没有初始化完成，但是第一个if条件这时候是成立的了，满足单例池中拿不出来和这个bean正在创建中，注意这时候beanName=“student”，然后它会先尝试去三级缓存earlySingletonObjects这个map当中去拿，好的大家看我前面，从来没有提起哪句代码将bean加入了这个map，所以肯定为空，那么他有什么作用呢，下面说，三级缓存拿不到，这时候去二级缓存拿，就是singletonFactories这个map，但是这个缓存拿出来是个工厂对象，没错这个当我们将bean加入这个map的时候，spring会把它作为参数传给工厂进行AOP的判断，所以这个工厂对象提供了getObject()方法去获取bean，所以从二级缓存拿分两步，先从map拿到工厂对象，然后这个工厂对象中再拿到bean，在拿到bean之后，spring做了一件事情，把这个bean加入三级缓存，再将这个bean从二级缓存中移除。那么问题来了，三级缓存到底有什么作用，明明一个二级缓存就可以解决所有的事情，干嘛需要这个三级缓存呢

这里我认为是这样的，singletonFactories这个map其实做了很多事情，比如AOP的相关操作等等，所以在这里如果我们没有这个earlySingletonObjects这个三级缓存，只有singletonFactories这个缓存在操作，我们试想一下，这个缓存拿出来是一个工厂对象，一个对象要在工厂里做很多事情，如果我们有100个类有A类的引用，A类有AOP的相关操作，那么singletonObject = singletonFactory.getObject();这个操作将要进行100次就是为了完成A类的AOP操作，所以会十分耗时，所以才有了earlySingletonObjects这个缓存，如果上面的操作singletonObject拿到了，那么将其加入earlySingletonObjects，从singletonFactories中移除，下次再依赖A，直接earlySingletonObjects中拿，代码中也是这么体现的，一定会先执行：singletonObject = this.earlySingletonObjects.get(beanName);到这里三级缓存就说完了，分别干了不同的事情

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && this.isSingletonCurrentlyInCreation(beanName)) {
            synchronized(this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = (ObjectFactory)this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }

        return singletonObject;
    }
```

## 四. spring-AOP

CGLIB动态代理与JDK动态区别：

JDK动态代理是利用反射机制生成一个实现代理接口的匿名类，在调用具体方法前调用InvokeHandler来处理。

而cglib动态代理是利用asm开源包，对代理对象类的class文件加载进来，通过修改其字节码生成子类来处理。

Spring中。

1、如果目标对象实现了接口，默认情况下会采用JDK的动态代理实现AOP

2、如果目标对象实现了接口，可以强制使用CGLIB实现AOP

3、如果目标对象没有实现了接口，必须采用CGLIB库，spring会自动在JDK动态代理和CGLIB之间转换

JDK动态代理只能对实现了接口的类生成代理，而不能针对类 。 CGLIB是针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法 。 因为是继承，所以该类或方法最好不要声明成final ，final可以阻止继承和多态。

> AOP的使用场景

* 日志记录
* 事务管理
* 性能监控
* 权限控制

> AOP源码分析

* @EnableAspectJAutoProxy给容器（beanFactory）中注册一个AnnotationAwareAspectJAutoProxyCreator对象
* AnnotationAwareAspectJAutoProxyCreator对目标对象进行代理对象的创建，对象内部，是封装JDK和CGlib两个技术，实现动态代理对象创建的（创建代理对象过程中，会先创建一个代理工厂，获取到所有的增强器（通知方法），将这些增强器和目标类注入代理工厂，再用代理工厂创建对象）
* 代理对象执行目标方法，得到目标方法的拦截器链，利用拦截器的链式机制，依次进入每一个拦截器进行执行

> CGLIB与JDK动态代理区别

- Jdk必须提供接口才能使用
- CGLIB不需要，只要一个非抽象类就能实现动态代理

