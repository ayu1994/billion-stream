基于kafka+ehcache+redis完成缓存数据生产服务的开发与测试

    1) 先是引入kafka包

    2) 注册监听器--InitListener

    3) 创建kafka的消费者线程

    4) 编写业务逻辑
       4.1两种服务会发送来数据变更消息：商品信息服务，商品店铺信息服务，每个消息都包含服务名以及商品id
       4.2接收到消息之后，根据商品id到对应的服务拉取数据，这一步，我们采取简化的模拟方式，就是在代码里面写死，会获取到什么数据，
          不去实际再写其他的服务去调用了
       4.3商品信息：id，名称，价格，图片列表，商品规格，售后信息，颜色，尺寸
       4.4商品店铺信息：其他维度，用这个维度模拟出来缓存数据维度化拆分，id，店铺名称，店铺等级，店铺好评率
       4.5分别拉取到了数据之后，将数据组织成json串，然后分别存储到ehcache中，和redis缓存中

    5) 测试
    （1）创建一个kafka topic
    （2）在命令行启动一个kafka producer
    （3）启动系统，消费者开始监听kafka topic
    （4）在producer中，分别发送两条消息，一个是商品信息服务的消息，一个是商品店铺信息服务的消息
    （5）能否接收到两条消息，并模拟拉取到两条数据，同时将数据写入ehcache中，并写入redis缓存中
    （6）ehcache通过打印日志方式来观察，redis通过手工连接上去来查询


总结

    缓存数据生产服务--->有数据变更---->主动更新两级缓存----->通过缓存维度化拆分
    分发层nginx+应用层nginx---->自定义流量分发策略提高缓存命中率
    nginx shard dict缓存 ----》缓存服务-----》redis----》ehcache---》渲染html模板----》返回页面
    
    
加入分布式锁功能
    
    1主动更新  --即监听kafka消息队列，获取到一个商品变更消息之后，去对应的源服务中调用接口拉取数据，
    更新到ehcache和redis中
    修改为：要先获取zookeeper分布式锁，然后才能更新redis，同时更新时要比较时间版本
    
    2 被动重建 --直接读取源头数据，然后返回给nginx，同时推送一条消息到一个队列，后台线程异步消费
    修改为后台线程负责先获取分布式锁，然后才能更新redis，同时要比较时间版本。