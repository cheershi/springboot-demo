# springboot启动原理（四）

AnnotationConfigEmbeddedWebApplicationContext这个类继承了EmbeddedWebApplicationContext类，GenericWebApplicationContext类(这个要注意)，它还实现了BeanDefinitionRegistry这个接口，还实现了ResourceLoader这个接口，这个类可以说是一个全能类了

refreshContext（）方法：


    public void refresh() throws BeansException, IllegalStateException {
        Object var1 = this.startupShutdownMonitor;
        synchronized(this.startupShutdownMonitor) {
            this.prepareRefresh();
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            this.prepareBeanFactory(beanFactory);

            try {
                this.postProcessBeanFactory(beanFactory);
                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                this.finishBeanFactoryInitialization(beanFactory);
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
 
 prepareRefresh方法
表示在真正做refresh操作之前需要准备做的事情：

设置Spring容器的启动时间，撤销关闭状态，开启活跃状态。

初始化属性源信息(Property)

验证环境信息里一些必须存在的属性


    protected void prepareRefresh() {
        //设置Spring容器的启动时间
        this.startupDate = System.currentTimeMillis();
        //撤销关闭状态
        this.closed.set(false);
        //开启活跃状态。
        this.active.set(true);
        if (this.logger.isInfoEnabled()) {
            this.logger.info("Refreshing " + this);
        }
         //初始化属性源信息(Property)
        this.initPropertySources();
        //验证环境信息里一些必须存在的属性
        this.getEnvironment().validateRequiredProperties();
        this.earlyApplicationEvents = new LinkedHashSet();
    }

prepareBeanFactory方法

从Spring容器获取BeanFactory(Spring Bean容器)并进行相关的设置为后续的使用做准备：

1.设置classloader(用于加载bean)，设置表达式解析器(解析bean定义中的一些表达式)，添加属性编辑注册器(注册属性编辑器)

2.添加ApplicationContextAwareProcessor这个BeanPostProcessor。取消ResourceLoaderAware、ApplicationEventPublisherAware、MessageSourceAware、ApplicationContextAware、EnvironmentAware这5个接口的自动注入。因为ApplicationContextAwareProcessor把这5个接口的实现工作做了

3.设置特殊的类型对应的bean。BeanFactory对应刚刚获取的BeanFactory；ResourceLoader、ApplicationEventPublisher、ApplicationContext这3个接口对应的bean都设置为当前的Spring容器

4.注入一些其它信息的bean，比如environment、systemProperties等

postProcessBeanFactory方法

BeanFactory设置之后再进行后续的一些BeanFactory操作。

不同的Spring容器做不同的操作。比如GenericWebApplicationContext容器会在BeanFactory中添加ServletContextAwareProcessor用于处理ServletContextAware类型的bean初始化的时候调用setServletContext或者setServletConfig方法(跟ApplicationContextAwareProcessor原理一样)。

AnnotationConfigEmbeddedWebApplicationContext对应的postProcessBeanFactory方法：

    @Override
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      // 调用父类EmbeddedWebApplicationContext的实现
      super.postProcessBeanFactory(beanFactory);
      // 查看basePackages属性，如果设置了会使用ClassPathBeanDefinitionScanner去扫描basePackages包下的bean并注册
      if (this.basePackages != null && this.basePackages.length > 0) {
        this.scanner.scan(this.basePackages);
      }
      // 查看annotatedClasses属性，如果设置了会使用AnnotatedBeanDefinitionReader去注册这些bean
      if (this.annotatedClasses != null && this.annotatedClasses.length > 0) {
        this.reader.register(this.annotatedClasses);
      }
    }
父类EmbeddedWebApplicationContext的实现：

    @Override
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
      beanFactory.addBeanPostProcessor(
          new WebApplicationContextServletContextAwareProcessor(this));
      beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    }

invokeBeanFactoryPostProcessors方法

在Spring容器中找出实现了BeanFactoryPostProcessor接口的processor并执行。Spring容器会委托给PostProcessorRegistrationDelegate的invokeBeanFactoryPostProcessors方法执行。

介绍两个接口：

1.BeanFactoryPostProcessor：用来修改Spring容器中已经存在的bean的定义，使用ConfigurableListableBeanFactory对bean进行处理

2.BeanDefinitionRegistryPostProcessor：继承BeanFactoryPostProcessor，作用跟BeanFactoryPostProcessor一样，只不过是使用BeanDefinitionRegistry对bean进行处理

基于web程序的Spring容器AnnotationConfigEmbeddedWebApplicationContext构造的时候，会初始化内部属性AnnotatedBeanDefinitionReader reader，这个reader构造的时候会在BeanFactory中注册一些post processor，包括BeanPostProcessor和BeanFactoryPostProcessor(比如ConfigurationClassPostProcessor、AutowiredAnnotationBeanPostProcessor)：

    AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
invokeBeanFactoryPostProcessors方法处理BeanFactoryPostProcessor的逻辑如下：

从Spring容器中找出BeanDefinitionRegistryPostProcessor类型的bean(这些processor是在容器刚创建的时候通过构造AnnotatedBeanDefinitionReader的时候注册到容器中的)，然后按照优先级分别执行，优先级的逻辑如下：

1.实现PriorityOrdered接口的BeanDefinitionRegistryPostProcessor先全部找出来，然后排序后依次执行

2.实现Ordered接口的BeanDefinitionRegistryPostProcessor找出来，然后排序后依次执行

3.没有实现PriorityOrdered和Ordered接口的BeanDefinitionRegistryPostProcessor找出来执行并依次执行

接下来从Spring容器内查找BeanFactoryPostProcessor接口的实现类，然后执行(如果processor已经执行过，则忽略)，这里的查找规则跟上面查找BeanDefinitionRegistryPostProcessor一样，先找PriorityOrdered，然后是Ordered，最后是两者都没。

这里需要说明的是ConfigurationClassPostProcessor这个processor是优先级最高的被执行的processor(实现了PriorityOrdered接口)。这个ConfigurationClassPostProcessor会去BeanFactory中找出所有有@Configuration注解的bean，然后使用ConfigurationClassParser去解析这个类。ConfigurationClassParser内部有个Map<ConfigurationClass, ConfigurationClass>类型的configurationClasses属性用于保存解析的类，ConfigurationClass是一个对要解析的配置类的封装，内部存储了配置类的注解信息、被@Bean注解修饰的方法、@ImportResource注解修饰的信息、ImportBeanDefinitionRegistrar等都存储在这个封装类中。

这里ConfigurationClassPostProcessor最先被处理还有另外一个原因是如果程序中有自定义的BeanFactoryPostProcessor，那么这个PostProcessor首先得通过ConfigurationClassPostProcessor被解析出来，然后才能被Spring容器找到并执行。(ConfigurationClassPostProcessor不先执行的话，这个Processor是不会被解析的，不会被解析的话也就不会执行了)。

在我们的程序中，只有主类RefreshContextApplication有@Configuration注解(@SpringBootApplication注解带有@Configuration注解)，所以这个配置类会被ConfigurationClassParser解析。解析过程如下：

1.处理@PropertySources注解：进行一些配置信息的解析

2.处理@ComponentScan注解：使用ComponentScanAnnotationParser扫描basePackage下的需要解析的类(@SpringBootApplication注解也包括了@ComponentScan注解，只不过basePackages是空的，空的话会去获取当前@Configuration修饰的类所在的包)，并注册到BeanFactory中(这个时候bean并没有进行实例化，而是进行了注册。具体的实例化在finishBeanFactoryInitialization方法中执行)。对于扫描出来的类，递归解析

3.处理@Import注解：先递归找出所有的注解，然后再过滤出只有@Import注解的类，得到@Import注解的值。比如查找@SpringBootApplication注解的@Import注解数据的话，首先发现@SpringBootApplication不是一个@Import注解，然后递归调用修饰了@SpringBootApplication的注解，发现有个@EnableAutoConfiguration注解，再次递归发现被@Import(EnableAutoConfigurationImportSelector.class)修饰，还有@AutoConfigurationPackage注解修饰，再次递归@AutoConfigurationPackage注解，发现被@Import(AutoConfigurationPackages.Registrar.class)注解修饰，所以@SpringBootApplication注解对应的@Import注解有2个，分别是@Import(AutoConfigurationPackages.Registrar.class)和@Import(EnableAutoConfigurationImportSelector.class)。找出所有的@Import注解之后，开始处理逻辑：

    1.遍历这些@Import注解内部的属性类集合
    
    2.如果这个类是个ImportSelector接口的实现类，实例化这个ImportSelector，如果这个类也是DeferredImportSelector接口的实现类，那么加入ConfigurationClassParser的deferredImportSelectors属性中让第6步处理。否则调用ImportSelector的selectImports方法得到需要Import的类，然后对这些类递归做@Import注解的处理
    
    3.如果这个类是ImportBeanDefinitionRegistrar接口的实现类，设置到配置类的importBeanDefinitionRegistrars属性中
    
    4.其它情况下把这个类入队到ConfigurationClassParser的importStack(队列)属性中，然后把这个类当成是@Configuration注解修饰的类递归重头开始解析这个类

4.处理@ImportResource注解：获取@ImportResource注解的locations属性，得到资源文件的地址信息。然后遍历这些资源文件并把它们添加到配置类的importedResources属性中

5.处理@Bean注解：获取被@Bean注解修饰的方法，然后添加到配置类的beanMethods属性中

6.处理DeferredImportSelector：处理第3步@Import注解产生的DeferredImportSelector，进行selectImports方法的调用找出需要import的类，然后再调用第3步相同的处理逻辑处理

这里@SpringBootApplication注解被@EnableAutoConfiguration修饰，@EnableAutoConfiguration注解被@Import(EnableAutoConfigurationImportSelector.class)修饰，所以在第3步会找出这个@Import修饰的类EnableAutoConfigurationImportSelector，这个类刚好实现了DeferredImportSelector接口，接着就会在第6步被执行。第6步selectImport得到的类就是自动化配置类。

EnableAutoConfigurationImportSelector的selectImport方法会在spring.factories文件中找出key为EnableAutoConfiguration对应的值，有81个，这81个就是所谓的自动化配置类(XXXAutoConfiguration)。

ConfigurationClassParser解析完成之后，被解析出来的类会放到configurationClasses属性中。然后使用ConfigurationClassBeanDefinitionReader去解析这些类。

这个时候这些bean只是被加载到了Spring容器中。下面这段代码是ConfigurationClassBeanDefinitionReader的解析bean过程：

    public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
      TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
      for (ConfigurationClass configClass : configurationModel) {
        // 对每一个配置类，调用loadBeanDefinitionsForConfigurationClass方法
        loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
      }
    }

    private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
        TrackedConditionEvaluator trackedConditionEvaluator) {
      // 使用条件注解判断是否需要跳过这个配置类
      if (trackedConditionEvaluator.shouldSkip(configClass)) {
        // 跳过配置类的话在Spring容器中移除bean的注册
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
          this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClassFor(configClass.getMetadata().getClassName());
        return;
      }

      if (configClass.isImported()) {
        // 如果自身是被@Import注释所import的，注册自己
        registerBeanDefinitionForImportedConfigurationClass(configClass);
      }
      // 注册方法中被@Bean注解修饰的bean
      for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
      }
      // 注册@ImportResource注解注释的资源文件中的bean
      loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
      // 注册@Import注解中的ImportBeanDefinitionRegistrar接口的registerBeanDefinitions
      loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }

invokeBeanFactoryPostProcessors方法总结来说就是从Spring容器中找出BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor接口的实现类并按照一定的规则顺序进行执行。 其中ConfigurationClassPostProcessor这个BeanDefinitionRegistryPostProcessor优先级最高，它会对项目中的@Configuration注解修饰的类(@Component、@ComponentScan、@Import、@ImportResource修饰的类也会被处理)进行解析，解析完成之后把这些bean注册到BeanFactory中。需要注意的是这个时候注册进来的bean还没有实例化。








