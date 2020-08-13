---
layout: post
title: SpringMvc 小结
category: Java
tags: []
keywords: 
---

#SpringMvc 小结/n    DispatcherServlet 前端控制器
    
    AbstractAnnotationConfigDispatcherServletInitializer 创建了 DispatcherServlet ContextLoaderListener 两个应用上下文
    
    GetServlet-ConfigClasses()方法返回的带有@Configuration注解的类将会用来定义DispatcherServlet应用上下文中的bean。
    
    getRootConfigClasses()方法返回的带有@Configuration注解的类将会用来配置ContextLoaderListener创建的应用上下文中的bean。
    
    
     