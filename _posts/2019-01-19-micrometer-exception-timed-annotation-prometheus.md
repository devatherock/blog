---
layout: post
title: "Exception when using Micrometer @Timed annotation on a @Scheduled method with PrometheusMeterRegistry and TimedAspect"
date: 2019-01-19
---
## The application

Consider the simple [spring boot](https://spring.io/projects/spring-boot){:target="_blank"} java app below:

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

import io.micrometer.core.annotation.Timed;
import io.micrometer.core.aop.TimedAspect;
import io.micrometer.core.instrument.MeterRegistry;

@SpringBootApplication
@EnableScheduling
public class MicrometerTimedAspectApp {
	private static final Logger LOGGER = LoggerFactory.getLogger(MicrometerTimedAspectApp.class);
	
	public static void main(String[] args) {
		SpringApplication.run(MicrometerTimedAspectApp.class, args);
	}
	
	@Bean
	public TimedAspect timedAspect(MeterRegistry meterRegistry) {
		return new TimedAspect(meterRegistry);
	}
	
	@Timed(value = "timer.sayHi", percentiles = { 0.50, 0.75, 0.95, 0.98, 0.99 })
	@Scheduled(fixedRateString = "10000")
	public String sayHi() {
		LOGGER.info("Saying Hi");
		return "hi";
	}
}
```
The `build.gradle` file for this application is as follows:
```groovy
plugins {
	id 'java'
	id 'org.springframework.boot' version '1.5.19.RELEASE'
}

repositories {
	jcenter()
}

dependencies {
	def springBootVersion = '1.5.19.RELEASE'
	def micrometerVersion = '1.0.7'
	
	compile group: 'org.springframework.boot', name: 'spring-boot-starter-web', version: springBootVersion
	compile group: 'org.springframework.boot', name: 'spring-boot-starter-actuator', version: springBootVersion
	compile group: 'io.micrometer', name: 'micrometer-core', version: micrometerVersion
	compile group: 'io.micrometer', name: 'micrometer-registry-prometheus', version: micrometerVersion
	compile group: 'io.micrometer', name: 'micrometer-spring-legacy', version: micrometerVersion
	compile group: 'org.aspectj', name: 'aspectjweaver', version: '1.9.2'
}
```

Notice that we are using the dependency `micrometer-registry-prometheus` which informs [micrometer](http://micrometer.io/){:target="_blank"} that the metrics backend we intend to use is [Prometheus](https://prometheus.io/){:target="_blank"}. 

## The Exception
Now run the application using `gradle bootrun`and monitor the logs.
You'll see an exception that a metric is being registered twice but with different set of tags:

```java
2019-01-19 10:07:59.278 ERROR 11088 --- [pool-1-thread-1] o.s.s.s.TaskUtils$LoggingErrorHandler    : Unexpected error occurred in scheduled task.

java.lang.IllegalArgumentException: Prometheus requires that all meters with the same name have the same set of tag keys. There is already an existing meter containing tag keys []. The meter you are attempting to register has keys [class, method].
        at io.micrometer.prometheus.PrometheusMeterRegistry.lambda$collectorByName$9(PrometheusMeterRegistry.java:360) ~[micrometer-registry-prometheus-1.0.7.jar:1.0.7]
        at java.util.concurrent.ConcurrentHashMap.compute(ConcurrentHashMap.java:1877) ~[na:1.8.0_191]
        at io.micrometer.prometheus.PrometheusMeterRegistry.collectorByName(PrometheusMeterRegistry.java:347) ~[micrometer-registry-prometheus-1.0.7.jar:1.0.7]
        at io.micrometer.prometheus.PrometheusMeterRegistry.newTimer(PrometheusMeterRegistry.java:160) ~[micrometer-registry-prometheus-1.0.7.jar:1.0.7]
        at io.micrometer.core.instrument.MeterRegistry.lambda$timer$2(MeterRegistry.java:258) ~[micrometer-core-1.0.7.jar:1.0.7]
        at io.micrometer.core.instrument.MeterRegistry.getOrCreateMeter(MeterRegistry.java:567) ~[micrometer-core-1.0.7.jar:1.0.7]
        at io.micrometer.core.instrument.MeterRegistry.registerMeterIfNecessary(MeterRegistry.java:529) ~[micrometer-core-1.0.7.jar:1.0.7]
        at io.micrometer.core.instrument.MeterRegistry.timer(MeterRegistry.java:256) ~[micrometer-core-1.0.7.jar:1.0.7]
        at io.micrometer.core.instrument.Timer$Builder.register(Timer.java:447) ~[micrometer-core-1.0.7.jar:1.0.7]
        at io.micrometer.core.aop.TimedAspect.timedMethod(TimedAspect.java:78) ~[micrometer-core-1.0.7.jar:1.0.7]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_191]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_191]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_191]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_191]
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:627) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:616) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:70) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:92) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:179) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:671) ~[spring-aop-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at com.devatherock.MicrometerTimedAspectApp$$EnhancerBySpringCGLIB$$e583a854.sayHi(<generated>) ~[main/:na]
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_191]
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_191]
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_191]
        at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_191]
        at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:65) ~[spring-context-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54) ~[spring-context-4.3.22.RELEASE.jar:4.3.22.RELEASE]
        at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_191]
        at java.util.concurrent.FutureTask.runAndReset(FutureTask.java:308) [na:1.8.0_191]
        at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_191]
        at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [na:1.8.0_191]
        at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_191]
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_191]
        at java.lang.Thread.run(Thread.java:748) [na:1.8.0_191]
```

The exception is because the [ScheduledMethodMetrics](https://github.com/micrometer-metrics/micrometer/blob/v1.0.7/micrometer-spring-legacy/src/main/java/io/micrometer/spring/scheduling/ScheduledMethodMetrics.java#L73-L75){:target="_blank"} aspect and [TimedAspect](https://github.com/micrometer-metrics/micrometer/blob/v1.0.7/micrometer-core/src/main/java/io/micrometer/core/aop/TimedAspect.java#L72-L78){:target="_blank"} are both trying to time the same method. `ScheduledMethodMetrics` creates timer `timer.sayHi` with no tags and then `TimedAspect` tries to create a timer with the same name but with additional `class` and `method` tags. The metric exposed by `http://localhost:8080/prometheus` actuator endpoint has no tags and looks like below:

```
# HELP timer_sayHi_seconds_max Timer of @Scheduled task
# TYPE timer_sayHi_seconds_max gauge
timer_sayHi_seconds_max 2.63195E-4
# HELP timer_sayHi_seconds Timer of @Scheduled task
# TYPE timer_sayHi_seconds summary
timer_sayHi_seconds{quantile="0.5",} 2.62144E-4
timer_sayHi_seconds{quantile="0.75",} 2.62144E-4
timer_sayHi_seconds{quantile="0.95",} 2.62144E-4
timer_sayHi_seconds{quantile="0.98",} 2.62144E-4
timer_sayHi_seconds{quantile="0.99",} 2.62144E-4
```

## The Workaround(s)
- Upgrading to micrometer `1.0.8` or above will get rid of the exception. This is due to a [catch](https://github.com/micrometer-metrics/micrometer/blob/v1.0.8/micrometer-core/src/main/java/io/micrometer/core/aop/TimedAspect.java#L80-L82){:target="_blank"} block added in TimedAspect to ignore any exception. However, in this case, the timer will not have the `class` or `method` tags added by the TimedAspect. Choose this workaround if you don't care about those tags
- Another workaround is to add the same class and method tags that will be added by the TimedAspect to the `@Timed` annotation

```java
@Timed(value = "timer.sayHi", percentiles = { 0.50, 0.75, 0.95, 0.98, 0.99 }, extraTags = { "class", 
	"com.devatherock.MicrometerTimedAspectApp", "method", "sayHi" })
@Scheduled(fixedRateString = "10000")
public String sayHi() {
	LOGGER.info("Saying Hi");
	return "hi";
}
```
In this case, there won't be any exception and the timer will have the class and method tags too.

```
# HELP timer_sayHi_seconds_max Timer of @Scheduled task
# TYPE timer_sayHi_seconds_max gauge
timer_sayHi_seconds_max{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",} 0.028372677
# HELP timer_sayHi_seconds Timer of @Scheduled task
# TYPE timer_sayHi_seconds summary
timer_sayHi_seconds{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",quantile="0.5",} 4.25984E-4
timer_sayHi_seconds{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",quantile="0.75",} 0.001359872
timer_sayHi_seconds{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",quantile="0.95",} 0.029343744
timer_sayHi_seconds{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",quantile="0.98",} 0.029343744
timer_sayHi_seconds{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",quantile="0.99",} 0.029343744
timer_sayHi_seconds_count{class="com.devatherock.MicrometerTimedAspectApp",method="sayHi",} 4.0
```
However in this case, the `timer_sayHi_seconds_count` count gets doubled. Choose this workaround if you are not relying on the count from this timer.

## Conclusion

The micrometer team is working on [rewriting](https://github.com/micrometer-metrics/micrometer/pull/500){:target="_blank"} the TimedAspect. Once that is complete, the workarounds won't be needed
