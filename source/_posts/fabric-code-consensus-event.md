---
title: fabric_code_consensus_event
categories:
  - fabric
date: 2019-4-14 15:07:25
tags:
---

## fabric consensus event源码解析

代码位置为 fabric/consensus/...

想写pbft的代码解析来着。看到里面的事件流，设计得很赞，学习一下。



#### Manager 事件管理

主要实现的功能是事件通道的处理，通过相关接口，写入数据到通道中，数据处理方提供处理方法即可，不需关注通道的实现。

```
//首先，接口事件要做的事主要是，内部提供一个事件通道，暴露出往通道写数据的接口，以及数据处理接收方法，接收方法由Receiver提供，Magager的实现者需要实现数据从通道读出来，回调给接收者的过程。
type Manager interface {
	Inject(Event)         //一个暂时的接口，跳过了通道，直接给到接收者，只限于Manager本线程使用，因为没有了通道做数据保护
	Queue() chan<- Event  //提供一个只写通道，用于往通道内写数据
	SetReceiver(Receiver) //设置接收者，接收者需要实现接收方法，在Receiver接口中有具体方法定义
	Start()               // 启动Manager线程
	Halt()                // 停止manager线程
}
```



##### Receiver接口

```
type Receiver interface {
	//事件处理的地方会调用这个方法传递事件给到Receiver,注意到这个设计的是返回值也是Event事件，这个在之后的地方进行详细解释
	ProcessEvent(e Event) Event
}
```



##### Manager实现

```
type managerImpl struct {
	threaded  //提供一个exit的通道，作为从外面结束线程的信号，包括Halt方法的实现
	receiver Receiver  //接收者
	events   chan Event //数据通道
}
```

之前说manager是一个事件通道的处理，那么，业务逻辑一定是不断循环，从通道中读出数据，然后调用接收者的方法传递给接收者。所以代码是..

```
func (em *managerImpl) Start() {
    //开启子线程
	go em.eventLoop()
}
func (em *managerImpl) eventLoop() {
	for {
		select {
		case next := <-em.events:
			em.Inject(next)
		case <-em.exit:
			logger.Debug("eventLoop told to exit")
			return
		}
	}
}
//传递Event给receiver
func (em *managerImpl) Inject(event Event) {
	if em.receiver != nil {
		SendEvent(em.receiver, event)
	}
}
//调用receiver.PeoceesEvent方法将事件传递给接收者
func SendEvent(receiver Receiver, event Event) {
	next := event
	for {
		// 如果ProcessEvent方法返回值不为空，则作为新的事件，继续处理
		next = receiver.ProcessEvent(next)
		if next == nil {
			break
		}
	}
}
```

ProcessEvent设计为有返回值，且为Event，当返回值不为空，则作为新的事件进行处理，直到返回值为空。当然，因为是同一个receiver，如果设计为返回值为空，然后在ProcessEvent里直接进行递归调用，感觉也是一样的。但这样写的话，在ProcessEvent就会干净很多。



#### Timer

go源码包中是有time包的，为什么还要封装一个timer呢，原因是封装的timer一旦被reset或stop，就算倒计时触发事件了，事件也不会传递到事件队列中。

既然提到go中原time包，多嘴一句，time包中的timer也是具体stop和reset方法，但可以看到官方函数解释

> ```
> Stop does not close the channel, to prevent a read from the channel succeeding incorrectly
> ```

即stop方法不会关闭通道，一般使用timer的定时器或ticker，都是监听通道是否有值，就意味着即使stop掉定时器，但通道还是在的，监听通道的程序不会退出。



好了，下面说明event中封装的Timer吧。

直接看实现吧，接口就略过了

```
// newTimer creates a new instance of timerImpl
func newTimerImpl(manager Manager) Timer {
	et := &timerImpl{
		startChan: make(chan *timerStart),
		stopChan:  make(chan struct{}),
		threaded:  threaded{make(chan struct{})},
		manager:   manager,
	}
	go et.loop()
	return et
}
```

注意到结构体中有manager的引用，实现的定时器为，当开启定时器时，需要传入定时时间，和event事件，当时间触发时，把event事件传递给manager.receiver，传递方式为manager中的传递方式。故主要代码为

```
func (et *timerImpl) loop() {
	var eventDestChan chan<- Event //内部缓存通道
	var event Event //事件

	for {
		select {
		case start := <-et.startChan:
			//计时开始，时间为start.duration，事件为start.event
			if et.timerChan != nil {
				if start.hard {
					logger.Debug("Resetting a running timer")
				} else {
					continue
				}
			}
			logger.Debug("Starting timer")
			et.timerChan = time.After(start.duration)
			if eventDestChan != nil {
				logger.Debug("Timer cleared pending event")
			}
			event = start.event
			eventDestChan = nil
		case <-et.stopChan:
			//结束计时
			if et.timerChan == nil && eventDestChan == nil {
				logger.Debug("Attempting to stop an unfired idle timer")
			}
			et.timerChan = nil
			logger.Debug("Stopping timer")
			if eventDestChan != nil {
				logger.Debug("Timer cleared pending event")
			}
			eventDestChan = nil
			event = nil
		case <-et.timerChan:
			//倒计时触发，这里好绕，倒计时触发，仅仅只是将事件传递通道的引用缓存下来
			logger.Debug("Event timer fired")
			et.timerChan = nil
			eventDestChan = et.manager.Queue()
		case eventDestChan <- event:
			//如果事件传递通道不为空，且event也不为空，则将event传递给事件通道中 eventDestChan <- event这一句话就竟然能实现这么多件事！
			logger.Debug("Timer event delivered")
			eventDestChan = nil
		case <-et.exit:
			//退出
			logger.Debug("Halting timer")
			return
		}
	}

```

代码看到，倒计时触发的时候，并不是立即直接将event送入通道，而是将通道缓存下来，等到下一次select，再执行将事件送入通道的事。**在nil通道上发送和接受将永远被阻塞，在select中，如果其通道是nil，它将永远不会被选择**。所以，上述eventDestChan如果为nil, case eventDestChan <- event的语句就不会被选择。eventDestChan不为nil时，就能被选择了，select的用法是不是感觉很赞！
