---
title: hystrix请求命令
date: 2019-06-16 18:52:14
tags:
---
Hystrix有两个请求命令 HystrixCommand、HystrixObservableCommand。

　　HystrixCommand用在依赖服务返回单个操作结果的时候。又两种执行方式

　　  -execute():同步执行。从依赖的服务返回一个单一的结果对象，或是在发生错误的时候抛出异常。

　　  -queue();异步执行。直接返回一个Future对象，其中包含了服务执行结束时要返回的单一结果对象。

　　HystrixObservableCommand 用在依赖服务返回多个操作结果的时候。它也实现了两种执行方式

　　  -observe():返回Obervable对象，他代表了操作的多个结果，他是一个HotObservable

　　  -toObservable():同样返回Observable对象，也代表了操作多个结果，但它返回的是一个Cold Observable。

# HystrixCommand：
**使用方式一：继承的方式**
```
package org.hope.hystrix.example;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixCommandKey;
import com.netflix.hystrix.HystrixRequestCache;
import com.netflix.hystrix.strategy.concurrency.HystrixConcurrencyStrategyDefault;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.client.RestTemplate;

/**
 * Created by lisen on 2017/12/15.
 * HystrixCommand用在命令服务返回单个操作结果的时候
 */
public class CommandHelloWorld extends HystrixCommand<String> {
    private final String name;

    public CommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected String run() throws Exception {
        int i = 1/0;
        return "Hello " + name + "!";
    }

    /**
     * 降级。Hystrix会在run()执行过程中出现错误、超时、线程池拒绝、断路器熔断等情况时，
     * 执行getFallBack()方法内的逻辑
     */
    @Override
    protected String getFallback() {
        return "faild";
    }
}
```
**单元测试：**
```
package org.hope.hystrix.example;

import org.junit.Test;
import rx.Observable;
import rx.Observer;
import rx.Subscription;
import rx.functions.Action1;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.Future;

import static org.junit.Assert.assertEquals;

/**
 * Created by lisen on 2017/12/15.
 *
 */
public class CommandHelloWorldTest {

    /**
     * 测试同步执行
     */
    @Test
    public void testSynchronous() {
        System.out.println(new CommandHelloWorld("World").execute());
    }

    /**
     * 测试异步执行
     */
    @Test
    public void testAsynchronous() throws ExecutionException, InterruptedException {
        Future<String> fWorld = new CommandHelloWorld("World").queue();
        System.out.println(fWorld.get());  //一步执行用get()来获取结果
    }

    /**
     * 虽然HystrixCommand具备了observe()和toObservable()的功能，但是它的实现有一定的局限性，
     * 它返回的Observable只能发射一次数据，所以Hystrix还提供了HystrixObservableCommand,
     * 通过它实现的命令可以获取能发多次的Observable
     */
    @Test
    public void testObserve() {
        /**
         * 返回的是Hot Observable,HotObservable，不论 “事件源” 是否有“订阅者”
         * 都会在创建后对事件进行发布。所以对于Hot Observable的每一个“订阅者”都有
         * 可能从“事件源”的中途开始的，并可能只是看到了整个操作的局部过程
         */
        //blocking
        Observable<String> ho = new CommandHelloWorld("World").observe();
//        System.out.println(ho.toBlocking().single());

        //non-blockking
        //- this is a verbose anonymous inner-class approach and doesn't do assertions
        ho.subscribe(new Observer<String>() {
            @Override
            public void onCompleted() {
                System.out.println("==============onCompleted");
            }

            @Override
            public void onError(Throwable e) {
                e.printStackTrace();
            }

            @Override
            public void onNext(String s) {
                System.out.println("=========onNext: " + s);
            }
        });

        ho.subscribe(new Action1<String>() {
            @Override
            public void call(String s) {
                System.out.println("==================call:" + s);
            }
        });
    }

    @Test
    public void testToObservable() {
        /**
         * Cold Observable在没有 “订阅者” 的时候并不会发布时间，
         * 而是进行等待，知道有 “订阅者” 之后才发布事件，所以对于
         * Cold Observable的订阅者，它可以保证从一开始看到整个操作的全部过程。
         */
        Observable<String> co = new CommandHelloWorld("World").toObservable();
        System.out.println(co.toBlocking().single());
    }

}
```
**使用方式二：注解的方式；**
```
package org.hope.hystrix.example.service;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.ObservableExecutionMode;
import com.netflix.hystrix.contrib.javanica.command.AsyncResult;
import org.springframework.stereotype.Service;
import rx.Observable;
import rx.Subscriber;

import java.util.concurrent.Future;

/**
 * 用@HystrixCommand的方式来实现
 */
@Service
public class UserService {
    /**
     * 同步的方式。
     * fallbackMethod定义降级
     */
    @HystrixCommand(fallbackMethod = "helloFallback")
    public String getUserId(String name) {
        int i = 1/0; //此处抛异常，测试服务降级
        return "你好:" + name;
    }

    public String helloFallback(String name) {
        return "error";
    }

    //异步的执行
    @HystrixCommand(fallbackMethod = "getUserNameError")
    public Future<String> getUserName(final Long id) {
        return new AsyncResult<String>() {
            @Override
            public String invoke() {
                int i = 1/0;//此处抛异常,测试服务降级
                return "小明:" + id;
            }
        };
    }

    public String getUserNameError(Long id) {
        return "faile";
    }
}
```
**单元测试：**
```
package org.hope.hystrix.example.service;

import javafx.application.Application;
import org.hope.hystrix.example.HystrixApplication;
import org.hope.hystrix.example.model.User;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.util.concurrent.ExecutionException;


@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = HystrixApplication.class)
public class UserServiceTest {

    @Autowired
    private UserService userService;

    /**
     * 测试同步
     */
    @Test
    public void testGetUserId() {
        System.out.println("=================" + userService.getUserId("lisi"));
    }

    /**
     * 测试异步
     */
    @Test
    public void testGetUserName() throws ExecutionException, InterruptedException {
        System.out.println("=================" + userService.getUserName(30L).get());
    }
}
```

# HystrixObservableCommand：

**使用方式一：继承的方式**

 　　HystrixObservable通过实现 protected Observable<String> construct() 方法来执行逻辑。通过 重写 resumeWithFallback方法来实现服务降级

```
package org.hope.hystrix.example;

import com.netflix.hystrix.HystrixCommandGroupKey;
import com.netflix.hystrix.HystrixObservableCommand;
import rx.Observable;
import rx.Subscriber;
import rx.schedulers.Schedulers;

public class ObservableCommandHelloWorld extends HystrixObservableCommand<String> {

    private final String name;

    public ObservableCommandHelloWorld(String name) {
        super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
        this.name = name;
    }

    @Override
    protected Observable<String> construct() {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                try {
                    if(!subscriber.isUnsubscribed()) {
                        subscriber.onNext("Hello");
                        int i = 1 / 0; //模拟异常
                        subscriber.onNext(name + "!");
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }

    /**
     * 服务降级
     */
    @Override
    protected Observable<String> resumeWithFallback() {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                try {
                    if (!subscriber.isUnsubscribed()) {
                        subscriber.onNext("失败了！");
                        subscriber.onNext("找大神来排查一下吧！");
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        }).subscribeOn(Schedulers.io());
    }
}
```
 **单元测试：**
```
package org.hope.hystrix.example;

import org.junit.Test;
import rx.Observable;

import java.util.Iterator;

public class ObservableCommandHelloWorldTest {

    @Test
    public void testObservable() {
        Observable<String> observable= new ObservableCommandHelloWorld("World").observe();

        Iterator<String> iterator = observable.toBlocking().getIterator();
        while(iterator.hasNext()) {
            System.out.println(iterator.next());
        }
    }

    @Test
    public void testToObservable() {
        Observable<String> observable= new ObservableCommandHelloWorld("World").observe();
        Iterator<String> iterator = observable.toBlocking().getIterator();
        while(iterator.hasNext()) {
            System.out.println(iterator.next());
        }
    }

}
```
**使用方式二：注解的方式；**
```
package org.hope.hystrix.example.service;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.ObservableExecutionMode;
import org.springframework.stereotype.Service;
import rx.Observable;
import rx.Subscriber;

@Service
public class ObservableUserService {
    /**
     *  EAGER参数表示使用observe()方式执行
     */
    @HystrixCommand(observableExecutionMode = ObservableExecutionMode.EAGER, fallbackMethod = "observFailed") //使用observe()执行方式
    public Observable<String> getUserById(final Long id) {
       return Observable.create(new Observable.OnSubscribe<String>() {
           @Override
           public void call(Subscriber<? super String> subscriber) {
               try {
                   if(!subscriber.isUnsubscribed()) {
                       subscriber.onNext("张三的ID:");
                       int i = 1 / 0; //抛异常，模拟服务降级
                       subscriber.onNext(String.valueOf(id));
                       subscriber.onCompleted();
                   }
               } catch (Exception e) {
                   subscriber.onError(e);
               }
           }
       });
    }

    private String observFailed(Long id) {
        return "observFailed---->" + id;
    }

    /**
     * LAZY参数表示使用toObservable()方式执行
     */
    @HystrixCommand(observableExecutionMode = ObservableExecutionMode.LAZY, fallbackMethod = "toObserbableError") //表示使用toObservable()执行方式
    public Observable<String> getUserByName(final String name) {
        return Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                try {
                    if(!subscriber.isUnsubscribed()) {
                        subscriber.onNext("找到");
                        subscriber.onNext(name);
                        int i = 1/0; ////抛异常，模拟服务降级
                        subscriber.onNext("了");
                        subscriber.onCompleted();
                    }
                } catch (Exception e) {
                    subscriber.onError(e);
                }
            }
        });
    }

    private String toObserbableError(String name) {
        return "toObserbableError--->" + name;
    }

}
```
**单元测试：**
```
package org.hope.hystrix.example.service;

import org.hope.hystrix.example.HystrixApplication;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;

import java.util.Iterator;

@RunWith(SpringJUnit4ClassRunner.class)
@SpringBootTest(classes = HystrixApplication.class)
public class ObservableUserServiceTest {

    @Autowired
    private ObservableUserService observableUserService;

    @Test
    public void testObserve() {
        Iterator<String> iterator = observableUserService.getUserById(30L).toBlocking().getIterator();
        while(iterator.hasNext()) {
            System.out.println("===============" + iterator.next());
        }
    }

    @Test
    public void testToObservable() {
        Iterator<String> iterator = observableUserService.getUserByName("王五").toBlocking().getIterator();
        while(iterator.hasNext()) {
            System.out.println("===============" + iterator.next());
        }
    }
}
```
**总结：**
　　在实际使用时，我们需要为大多数执行过程中可能会失败的Hystrix命令实现服务降级逻辑，但是也有一些情况可以不去实现降级逻辑，比如：

　　**执行写操作的命令：**

　　**执行批处理或离线计算的命令：**
