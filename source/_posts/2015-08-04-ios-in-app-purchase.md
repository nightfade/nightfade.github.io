layout: post
date: 2015-08-09 04:55:12
title: iOS应用内购买IAP接入
tags: 
- iOS
---

本文主要讲解iOS应用接入应用内购买的基本实现。
首先需要在应用后台做一些配置工作，这部分就不做详细步骤的说明，只列出需要完成的工作：
1. 在[developer.apple.com](https://developer.apple.com/)为应用创建APP ID。
2. 在[iTunes Connect](https://itunesconnect.apple.com/)中，创建一个新的应用。
3. 在该应用中，创建应用内付费项目，然后设置好价格和`ProductID`，以及其他相关信息，这里的`ProductID`在之后的开发中会用到。
4. 添加一个`Test User`用作之后`SandBox`付费的测试用户。

配置工作完成后，就进入开发阶段。
IAP的接入依赖于`StoreKit.framework`，因此需要在工程中引入`StoreKit.framework`。
相关的几个主要对象有：
- `SKProduct`：商品信息，包括价格、商品名称、描述等。
- `SKProductRequest`：发起网络请求，从`App Store`获取商品信息。
- `SKPaymentQueue`：支付执行队列，用来发起支付操作。
- `SKPaymentTransaction`：交易对象，每次支付交易对应一个交易对象。

IAP的基本流程如下。
首先，根据之前配置好的`ProductID`，这里假定为`com.demo.product`，从`App Store`获取商品信息，即`SkProduct`对象：

```objc
// SKProductRequest * _productsRequest;
- (void)requestProducts {
    NSSet * productIdentifiers = [NSSet setWithObjects:@"objc", nil];
    _productsRequest = [[SKProductsRequest alloc] initWithProductIdentifiers:productIdentifiers];
    _productsRequest.delegate = self;
    [_productsRequest start];
}
```

通过`SKProductRequest`获取商品信息成功/失败后，会调用该对象的`delegate`(实现了`<SKProductsRequestDelegate>`)的回调：
```objc
#pragma mark - SKProductsRequestDelegate
- (void)productsRequest:(SKProductsRequest *)request didReceiveResponse:(SKProductsResponse *)response {
    NSLog(@"Loaded list of products...");
    _productsRequest = nil;
    
    NSArray * skProducts = response.products;
    for (SKProduct * skProduct in skProducts) {
        NSLog(@"Found product: %@ %@ %@ %0.2f",
              skProduct.productIdentifier,
              skProduct.localizedTitle,
              skProduct.priceLocale,
              skProduct.price.floatValue);
    }
}

- (void)request:(SKRequest *)request didFailWithError:(NSError *)error {
    NSLog(@"Failed to load list of products.");
    _productsRequest = nil;
}
```

如果请求成功，我们就已经得到了`SKProduct`对象，接下来可以发起购买请求：
```objc
- (void)buyProduct:(SKProduct *)product {
    NSLog(@"Buying %@...", product.productIdentifier);
    SKPayment * payment = [SKPayment paymentWithProduct:product];
    [[SKPaymentQueue defaultQueue] addPayment:payment];
}
```
`SKPaymentQueue`处理购买请求的结果是通过回调`Observer`（实现了`<SKPaymentTransactionObserver>`）来通知的。这里需要处理的两种可能情况：
1. 当下发起的购买请求，保持程序运行状态，最终等到返回的购买结果。
2. 发起购买请求后，尚未等到返回的购买结果之前，就退出了程序（主动退出，或者程序崩溃）。

对于第二种情况，`StoreKit`的处理逻辑是，在程序启动后，为标记完成的`Transaction`会重新回调`Observer`进行处理。因此需要保证在程序启动时注册`SKPaymentQueue`的`Observer`。因此我们在`- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions;`中完成这一操作：
```objc
[[SKPaymentQueue defaultQueue] addTransactionObserver:observer];
```
并在`Observer`中实现相关回调：
```objc
#pragma mark - SKPaymentTransactionObserver
- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray *)transactions {
    for (SKPaymentTransaction * transaction in transactions) {
        switch (transaction.transactionState) {
            case SKPaymentTransactionStatePurchased:
                [self completeTransaction:transaction];
                break;
            case SKPaymentTransactionStateRestored:
                [self restoreTransaction:transaction];
                break;
            case SKPaymentTransactionStateFailed:
                [self failedTransaction:transaction];
                break;
            default:
                break;
        }
    }
}

#pragma mark - Callbacks
- (void)completeTransaction:(SKPaymentTransaction *)transaction {
    NSLog(@"complete transaction...");
    // TODO: verify transaction receipts, provide contents
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}

- (void)restoreTransaction:(SKPaymentTransaction *)transaction {
    NSLog(@"restore transaction...");
    // TODO: verify transaction receipts, provide contents
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}

- (void)failedTransaction:(SKPaymentTransaction *)transaction {
    NSLog(@"failed transaction...");
    [[SKPaymentQueue defaultQueue] finishTransaction:transaction];
}
```
之前提到，如果没有将`Transaction`标记完成，每次程序启动后，都会调用`Observer`的回调继续处理该`Transaction`。因此在交易完成，已经提供给用户商品或者确认交易已经失败之后调用:
```objc
[[SKPaymentQueue defaultQueue] finishTransaction:transaction];
```
讲交易标记成完成。

另外这里还需要注意的一点是，考虑到伪造交易`Receipt`的问题，如果应用有服务端，可以考虑将`Receipt`传给服务端，并由服务端向`App Store`通过`HTTP API`发起`Receipt`验证请求，以`Node.js`为例：
```javascript
var makeRequest = require('request');

makeRequest({
        url: 'https://buy.itunes.apple.com/verifyReceipt',
        method: 'POST',
        json: {'receipt-data':receipt},
        headers: {"content-type": "application/json"}
    }, function (err, httpResponse, body) {
        if (err) {
            console.log('request error: ' + err.message);
        } else if (body.status === 0) {
            console.log('valid receipt');
        } else {
            console.log('invalid receipt');
        }
    });
```
`App Store`正式环境验证URL是`https://buy.itunes.apple.com/verifyReceipt` ，测试环境验证URL是：`https://sandbox.itunes.apple.com/verifyReceipt`。

对于非消耗商品（Non-Consumable），Apple要求必须实现`Restore`功能，即用户在其他设备上购买的商品，更换设备后，可以在新的设备上直接取回已购的商品。实现`Restore`很简单：
```objc
[[SKPaymentQueue defaultQueue] restoreCompletedTransactions];
```
该操作完成后也同样会调用`<SKPaymentTransactionObserver>`的
`- (void)paymentQueue:(SKPaymentQueue *)queue updatedTransactions:(NSArray *)transactions;`回调，且`transaction.transactionState == SKPaymentTransactionStateRestored`，可由此触发我们自己的`Restore`逻辑。

最后，由于IAP测试的特殊性，还有一些问题需要注意。
1. `Simulator`上无法完成IAP交易，无论是沙盒环境还是正式环境都不行。
2. `Developer`证书 + 测试账号触发沙盒环境，`Distribution`证书触发正式环境。要触发沙盒环境，首先在iOS的`Setting`中登出当前账号，之后在应用内付费弹出输入App Id的时候输入测试账号。
3. 测试账号一定不要用来登录正式环境，否则该测试账号再也不能用来登录沙盒环境做测试。
4. 在提交Apple审核时，启用的是App自己服务端的正式环境和Apple的**沙盒环境**，因此对于我们自己服务端正式环境来说，如果有验证`Receipt`的逻辑，建议的实现方式是：先走`App Store`正式环境的URL验证`Receipt`，如果`status`返回值是`21007`(沙盒环境的`Receipt`拿到了正式环境做验证)，再向沙盒环境URL重新做一次验证，从而保证应用审核顺利进行。