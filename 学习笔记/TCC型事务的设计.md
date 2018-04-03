基于注解的TCC型最终一致性事务的设计：

可以用`global_transaction`表示一个全局事务的开始，`global_transaction_process`表示一个全局事务的子步骤。

我们用一个全局事务表来记录一个全局事务，用一个全局事务步骤表来记录全局事务步骤。

我们分别需要拦截这两个注解来生成各个子步骤:

1. 当我们拦截到`global_transaction`时，创建一个全局事务，传递一个标志标明调用的方法是try方法，设置事务状态为TRY,并在调用上下文中传递这个全局事务。

````
@global_transaction
public void makePayment(Order order, BigDecimal redPacketPayAmount, BigDecimal capitalPayAmount) {
    System.out.println("order try make payment called");

    order.pay(redPacketPayAmount, capitalPayAmount);
    orderRepository.updateOrder(order);

    String result = capitalTradeOrderService.record(buildCapitalTradeOrderDto(order));
    String result2 = redPacketTradeOrderService.record(buildRedPacketTradeOrderDto(order));
}
````

2. 当我们拦截到`global_transaction_process`时，如果调用上下文中有全局事务标志，并且全局事务状态为TRY,那么就为本次调用创建一个全局事务子步骤，并且记录此次子事务的commit方法和cancell方法，在全局事务步骤表中添加一条记录，并且子步骤状态为TRY。

3. 经过上述步骤，我们就可以为一个全局事务在全局事务步骤表中记录所有的步骤，并且状态为TRY。

当TRY过程都成功时，调用所有子事务的commit方法。如果所有的子事务都commit成功，那么我们就完成这次事务，否则，就调用cancell方法。

4. 当我们调用全局事务发起接口的commit方法时，实际上调用的是一个被代理的方法。此时，我们将事务状态改为COMMIT,把这个事务中所有的子步骤load出来，分别调用它们的commit方法。

5. cancell与步骤4相同。分别调用子步骤的cancell方法。

可以为每个TCC事务创建一个事务管理器类，事务管理器类里用一个list装本次事务的所有子事务和全局事务。
