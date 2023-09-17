> Spring源码版本为v5.2.6.RELEASE
### XmlBeanFactory的工作原理分析
![XmlBeanFactory](../images/spring/2023-09-18_XmlBeanFactory的工作原理分析.png ':size=60%')  
首先，通过ClassPathResource将applicationContext.xml配置文件封装起来，ClassPathResource会从resources目录下解析配置文件，从配置文件中解析bean标签，并获取bean标签上的id属性和class属性的值。通过class属性的值(即类全限定名称)，就可以通过反射创建bean，也就是创建了一个Player对象出来，然后再将Player对象放到Spring容器当中，Player对象在容器中的名称为属性id的值，Spring容器的初始化简单来说也就是干这些事。  
然后，当调用getBean方法时就会从Spring容器中加载bean了，Spring会根据给定bean的名称到Spring容器中获取bean，比如图中就是通过Player这个名称从Spring容器中获取Player对象。