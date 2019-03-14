##Android聊天消息高并发优化

> 随着cctalk的业务量增加，从私聊到群，再从群到万人群。聊天消息量的爆增随之而来的就是本地聊天消息处理能力的瓶颈影现。高并发下聊天消息的优化需求迫切

### 当前处理能力
按照持续50条/s的消息频率大概在处理1.5w条消息后，前台刷新数据量固定为缓冲数据块大小。说明有消息堆积每次缓冲都是满负荷处理。

如何提高处理能力
---
### 1.建立缓冲buffer
聊天消息需要本地存储，数据库肯定是最耗时的。翻看代码发现聊天消息数据是单条插入的，并且为了避免重复消息每次都执行两步操作(删除相同数据，插入新的数据)。所以第一改造目标就是**批量处理数据**。

####如何批量
1. 服务器打包多条消息发送
2. 本地建立缓冲池收集消息批量处理

服务器对聊天消息的发送有一定的优化，但打包消息发送短期内实现不现实。所以只能走第二条路线。

####怎么建立缓冲
缓冲池目的就是时间片段（timespan）内不会给上层业务数据。时间片段内的量是不可控的，最好有一个上限（count）。timespan和count是2个关键点。项目中使用了rxjava所以利用rxjava能很快方便的实现这个缓冲池

```
	private Subject<MessageVo> publisher = PublishSubject.<MessageVo>create().toSerialized();
	publisher.toFlowable(BackpressureStrategy.BUFFER)
                .groupBy(new Function<MessageVo, Long>() {
                    @Override
                    public Long apply(@NonNull MessageVo messageVo) throws Exception {
                        return messageVo.getSubjectId();
                    }
                })
                .subscribeOn(HJExecutorManager.executeScheduler())
                .observeOn(HJExecutorManager.executeScheduler())
                .subscribe(new Consumer<GroupedFlowable<Long, MessageVo>>() {
                    @Override
                    public void accept(@NonNull final GroupedFlowable<Long, MessageVo> longMessageVoGroupedFlowable) throws Exception {
                        longMessageVoGroupedFlowable
                                .buffer(TIME_SPAN, TimeUnit.MILLISECONDS, MAX_COUNT)
                                .subscribe(new Consumer<List<MessageVo>>() {
                                    @Override
                                    public void accept(@NonNull List<MessageVo> messageVos) throws Exception {
                                        if (messageVos.size() == 0) {
                                            return;
                                        }
                                        //上层接口处理缓冲数据
                                        MessageBuffer.this.listener.buffer(longMessageVoGroupedFlowable.getKey(), messageVos);
                                    }
                                });
                    }
                });
                
```

###2.sqlite事务批量处理
通过sqlite事务批量读写数据库会比单条数据循环读写带来的行能提升很明显

```
		try {
            writedb.beginTransaction();
            for (MessageVo messageVo : messageVoList) {
                //db insert ...
            }
            writedb.setTransactionSuccessful();
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            writedb.endTransaction();
        }
        
```

###3.日志耗时
通过上两步的优化处理能力有所提升，但实际体验效果没有提升很多。大概在2w多条的时候又有消息堆积了。再次翻看源码考虑从业务逻辑代码性能方面的优化，检查业务逻辑是否有效率低，性能差的代码实现。但是这块代码业务复杂，时间悠久，如果优化这个点就相当于重构消息处理实现，从架构层次考虑风险很大，涉及项目模块多，短期根本不可能成功。并且这些业务不能减少，即使重构成功性价比不高。源代码已经不能看出优化点了，只能通过性能工具检测。通过traceview检测发现cpu消耗时间里面有一个String.fromat耗时占30%。继续跟进代码发现每条最原始的聊天消息都会被log日志打印出来，而这个耗时就是log日志中做了format。这个很不起眼的日志尽然占用了30%的执行时间。这个坑会被很多人遗忘，但这绝对是优化上的性价比最高的点

###4.delete update需要建立索引
traceview显示log日志耗时30%，另外还有一个耗时点发现在database.delete()方法上。一开始想删除一条数据为什么会很耗时，并且这些删除都放在了事务里面。细看代码发现删除消息的sql语句包含了5个条件的where语句。后来一想会不会这个where语句很耗时，然后对这个where语句建立了一个索引，重新跑起程序，发现效果显著提升。持续300条/s的频率跑起来毫无延迟感。通过这个优化可以发现在DB处理大数据量的时候如果有delete,update等where条件相关的操作时一定要对where条件的column建立索引。

###总结
在大并发的场景下不仅仅是数据库会造成瓶颈，无用的日志也可能埋下坑而且不容易被发现。同时缓冲池的不是一成不变的。要结合实际的场景调到最佳的比。