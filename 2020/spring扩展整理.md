# Bean装载过程中触发

## 		BeanNameAware　获取bean的id

## 		ApplicationContextAware　获取applicationContext

## 		BeanFactoryAware  获取beanFactory

## 		FactoryBean 实现这个接口以后每次自动装配都会进入getObject方法

## 		BeanPostProcessor　每bean装载前后都会调用

## 		BeanFactoryPostProcessor修改bean定义或新建bean定义，只会执行一次

# Bean装载完成时触发

## 		SmartLifecycle Bean 初始化完毕后调用．可以用于开启定时任务

# Spring事件回调



# Spring Cloud网关过滤实现

## 		GlobalFilter