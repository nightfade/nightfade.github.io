layout: post
date: 2015-08-12 20:29
tags: 
- iOS
title: iOS开发中的Promise模式
---

# 什么是Promise
借用[`PromiseKit`](http://promisekit.org)文档中的定义：
> A promise represents the future value of an asynchronous task.

如果去Google一下`Promise模式`关键字，会发现基本都是和`JavaScript`相关的，在`JavaScript`中，`Promise`模式应用的非常多，因为`JavaScript`语言自身的特点（函数是一等对象），以及运行环境（浏览器，Node.js）决定了在`JavaScript`里异步操作运用的非常多，所以很多`JavaScript`开发者也想了很多办法更好的管理这些异步操作，`Promise`模式是其中最简洁有效的一种。

在iOS开发中，同样会面临和`JavaScript`运行环境类似的问题。作为客户端开发，我们必须要保证用户界面流畅不卡顿，所以要尽量避免有长时间运行的操作阻塞主线程。那使用的方式就是把耗时比较长的比如I/O操作、网络请求、数据库查询之类的操作放到后台线程里去做，等操作完成之后再通知主线程更新UI。这些放到后台里任务就构成了一系列异步操作。最传统的处理异步任务的方式是使用回调(`Callback`)。回调本身没什么问题，但是使用回调的异步操作代码相对来说更难看了。

编造一个真实场景的例子，假设我们要处理一个用户登录的流程：
1. 首先向服务器发起网络请求验证用户的账号密码。
2. 验证成功之后查询本地数据库里有没有用户的相关信息：
    a. 如果没有，向服务器发起网络请求拉取用户信息.
    b. 如果有，把本地用户数据的时间戳发送给服务器检查需不需要更新数据，如果需要的话同样也做一次数据拉取。
3. 拉取用户信息成功之后，把拉取到用户信息写回到数据库。

整个流程中还需要考虑每一步的错误处理。

如果不考虑主线程阻塞的问题，就可以这么写：
```objc
- (void)doSync { 
    BOOL loginSuccess = [_client loginWithUsername:@"aUser" andPassword:@"123456"];
    if (!loginSuccess) {
        NSLog(@"登录验证失败！");
    }
    id userData = [_db queryWithUsername:@"aUser"];
    BOOL needFetch = NO;
    if (userData) {
        NSNumber * timestamp = [userData timestamp];
        needFetch = [_client checkTimestamp:timestamp];
    }
    if (needFetch) {
        id userData = [_client fetchUserData];
        [_db writeUserData:userData];
    }
    NSLog(@"操作成功");
}
```
整个流程写下来非常清晰。但是真实情况下，为了避免阻塞线程，我们都会把网络操作、I/O操作包括数据库操作等等写成异步接口，这种情况下我们再看下代码要怎么写：
```objc
- (void)doAsync {
    [_client loginWithUsername:@"aUser" andPassword:@"123456" result:^(BOOL success){
        if (success) {
            [_db queryWithUsername:@"aUser" result:^(id userData){
                if (userData) {
                    NSNumber * timestamp = [userData timestamp];
                    [_client checkTimestamp:timestamp result:^(BOOL needFetch){
                        if (needFetch) {
                            [_client fetchUserDataWithResult:^(id userData){
                                [_db writeUserData:userData result:^{
                                    NSLog(@"操作成功!");                                
                                }]                            
                            }];
                        } else {
                            NSLog(@"操作成功!");
                        }
                    }];                
                }            
            }];
        } else {
            NSLog(@"登录失败！");
        }
    }];
}
```
最后再看一下用了`Promise`模式的异步接口，代码是什么样子：
```objc
- (void)doPromiseAsync {
    PMKPromise *loginPromise = [_client loginWithUsername:@"aUser" andPassword:@"123456"];
    PMKPromise * queryPromise = loginPromise.then(^{
        return [_db queryWithUsername:@"aUser"];    
    });
    PMKPromise * checkPromise = queryPromise.then(^(id userData){
        if (userData) {
            NSNumber * timestamp = [userData timestamp];
            return [_client checkTimestamp:timestamp];
        } else {
            return YES;
        }
    });
    PMKPromise * needFetchPromise = checkPromise.then(^(BOOL needFetch){
        if (!needFetch) {
            return;
        }
        PMKPromise * fetchPromise = [_client fetchUserData];
        PMKPromise * writePromise = fetchPromise.then(^(id userData){
            return [_db writeUserData:userData];    
        });
    });
    needFetchPromise.then(^{
        NSLog(@"操作成功！");
    }).catch(^(NSError *err){
        NSLog(@"操作失败：%@", err);
    });
}
```
使用`Promise`构建的异步操作的代码结构变得和同步接口很相似了，调用异步接口直接通过返回值返回一个`Promise`对象，这个对象‘保证‘会把相应异步操作会在将来到来的结果的值，通过`then`方法交给后续的调用。`Promise`支持链式调用，所以上面的例子中，我们可以省略中间变量，写成下面这种更简洁的方式：
```objc
- (void)doPromiseAsync {
    [_client loginWithUsername:@"aUser" andPassword:@"123456"].then(^{
        return [_db queryWithUsername:@"aUser"];    
    }).then(^(id userData){
        if (userData) {
            NSNumber * timestamp = [userData timestamp];
            return [_client checkTimestamp:timestamp];
        } else {
            return YES;
        }
    }).then(^(BOOL needFetch){
        if (!needFetch) {
            return;
        }
        return [_client fetchUserData].then(^(id userData){
            return [_db writeUserData:userData];    
        });
    }).then(^{
        NSLog(@"操作成功！");
    }).catch(^(NSError *err){
        NSLog(@"操作失败：%@", err);
    });
}
```
简而言之，使用`Promise`模式写异步操作，避免了层层嵌套的回调，使得代码结构和逻辑更直接更清晰。像使用同步接口一样，代码的书写顺序就是代码的执行顺序。

# PromiseKit的基本使用
直接引用[`PromiseKit`](http://promisekit.org)首页的说明：
> PromiseKit is not just a promises implementation, it is also a collection of helper functions that make the typical asynchronous patterns we use as iOS developers delightful too.

首先看一下`Promise`对象`PMKPromise`的主要接口：
## `then`
`then`用来传入异步操作完成之后对结果的操作的Block。例如：
```objc
UIAlertView *alertView = [UIAlertView …];
[alertView promise].then(^(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
});
```
Block的返回值可以是任意对象类型，值类型，`PMKPromise`对象类型，或者`NSError`类型。`then`的返回值也是`PMKPromise`类型。为了方便描述，假设原`PMKPromise`对象是`A`，`A`的`then`方法返回的`PMKPromise`对象是`B`。
- 如果Block的返回值是对象类型或者值类型，那么`B`也会进一步向后传递这个值或者对象给`then`的Block。
- 如果Block的返回值也是`PMKPromise`对象，称作`C`，那么会首先执行`C`对应的异步操作，再把将`C`的Block的返回值传递给`B`。
- 如果Block的返回值类型是`NSError`，那么将触发错误处理流程。这部分在错误处理的章节会做详细说明。

举例说明：
1. Block返回值是值/对象类型：
```objc
[alertView promise].then(^(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
    return self.contacts[dismissedIndex.integerValue];
}).then(^(id contact){
    NSLog(@"contact name: %@ pressed.", [contact name]);
});
```
2. Block返回值类型也是`PMKPromise`类型：
```objc
[alertView promise].then(^(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
    return [self.client promiseQueryCotactAtIndex:dismissedIndex];
}).then(^(id contact){
    NSLog(@"contact name: %@ pressed.", [contact name]);
});
```
## `catch`
如果调用链上某个Block返回值类型是`NSError`类型，`NSError`对象会直接跳过调用链上的`then`方法向下传递，直到遇到`catch`方法进行处理：
```objc
[alertView promise].then(^id(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
    if (dismissedIndex < self.contacts.count){
        return self.contacts[dismissedIndex.integerValue];
    } else {
        return [NSError errorWithDomain:NSDemoErrorDomain code:123 userInfo:nil];
    }
}).then(^(id contact){
    NSLog(@"contact name: %@ pressed.", [contact name]);
}).catch(^(NSError *err){
    NSLog(@"error occurred with code: %@", @(err.code));    
});
```
`catch`方法除了可以进行错误处理，甚至还可以进行错误恢复。如果`catch`方法有非`NSError`的返回值，那就暗示着错误已经被处理了，所以后续的调用链会继续执行：
```objc
[alertView promise].then(^id(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
    if (dismissedIndex < self.contacts.count){
        return self.contacts[dismissedIndex.integerValue].name;
    } else {
        return [NSError errorWithDomain:NSDemoErrorDomain code:123 userInfo:nil];
    }
}).catch(^(NSError *err){
    if (err.code == 123) {
        return @"Unknown Name";
    } else {
        // return err;
        @throw err;
    }
}).then(^(NSString * contactName){
    NSLog(@"contact name: %@ pressed.", contactName);
}).catch(^(NSError *err){
    NSLog(@"fatal error: %@", @(err.code));    
});
```
## finally
`finally`是`then`和`catch`的补充，无论调用链最终执行`then`还是`catch`，`finally`的代码块始终都会得到执行：
```objc
[alertView promise].then(^id(NSNumber *dismissedIndex){
    NSLog(@"index %@ pressed.", dismissedIndex);
    if (dismissedIndex < self.contacts.count){
        return self.contacts[dismissedIndex.integerValue];
    } else {
        return [NSError errorWithDomain:NSDemoErrorDomain code:123 userInfo:nil];
    }
}).then(^(id contact){
    NSLog(@"contact name: %@ pressed.", [contact name]);
}).catch(^(NSError *err){
    NSLog(@"error occurred with code: %@", @(err.code));    
}).finally(^{
    NSLog(@"operation completed.");
});
```

了解了`PMKPromise`的主要对象，我们看一下怎么封装我们自己的`Promise`模式的异步API：
```objc
- (PMKPromise *)users {
    return [PMKPromise promiseWithResolverBlock:^(PMKResolver resolve) {
        AVQuery *query = [AVUser query];
        [query findObjectsInBackgroundWithBlock:^(NSArray *objects, NSError *error) {
            resolve(error ?: objects);
        }];
    }];
}
```
`PMKPromise`对象的`promiseWithResolverBlock`方法会传入的`resolve`参数，我们可以在异步操作完成的时候，讲异步操作的结果传给`resolve`。如果传入的是非`NSError`对象，那么就触发后续的`then`，否则触发后续的`catch`。

# PromiseKit进阶
## Dispatch Queues
现在我们来讨论一下`PromiseKit`与`GCD`。之前我们看到的`PMKPromise`的`then`方法默认都是执行在主线程上的，但是我们可以很容易的通过`thenOn`方法把这些代码的执行`dispatch`到其他线程，例如：
```objc
id url = @"http://www.baidu.com/test.png";
id q = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, 0);
[NSURLConnection GET:url].thenOn(q, ^(UIImage *image){
    assert(![NSThread isMainThread]);
});
```
`PromiseKit`方法还提供了`thenInBackground`的便利函数，把后续的执行`dispatch`到`GCD`的`DISPATCH_QUEUE_PRIORITY_DEFAULT`队列。

## `when`
之前我们例子里通过`then`去串行执行异步操作。很多时候我们还需要并行执行异步操作，等所有异步操作都执行完毕以后再继续后续操作。这时候就用到`when`方法，举例说明：
```objc
id search1 = [[[MKLocalSearch alloc] initWithRequest:rq1] promise];
id search2 = [[[MKLocalSearch alloc] initWithRequest:rq2] promise];
// PMKWhen(@[a, b]) 等价于 [PMKPromise when:@[a, b]]
PMKWhen(@[search1, search2]).then(^(NSArray *results){
    id resultOfSearch1 = results[0];
    id resultOfSearch2 = results[1];
}).catch(^(NSError *error){
    // called if either search fails
});
```

至此，`PromiseKit`的常用的方法基本就是这些，其他更多的用法，可以参考[`PromiseKit`文档](http://promisekit.org)。