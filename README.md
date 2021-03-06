一、	方案原则
1、尽量使用现有的成熟的，经过验证的技术组件。数据分离做到稳定可控。
2、冷热数据对业务层透明，对上层业务只暴露redis接口。尽量做到不需要改变目前的开发习惯。
3、兼顾新项目与老项目，新项目用此方案不会block开发进度，老项目用此方案不会因为数据分离本身承担数据丢失，停机等风险。
4、数据分离之后性能损失很小，访问冷数据的业务不会影响到访问热数据业务的处理速度。
5、易扩容
二、	数据层架构

图一（数据层总框架）

 
图二（落地层框架）
1、	热数据放redis集群中
Redis集群保持目前的数据可靠性(3副本集、定时做数据落地)。
	老项目方案
老项目就按目前数据存储方式不需要改变。
	新项目方案
虽然把redis看成一个大的集群，但是需要对每数据具体存放在哪个redis上做好数据隔离，与风险管控,避免因为单节点故障出现全区全局掉线等问题。比如一个玩家的数据就按玩家的id对数据进行路由，放同一个redis中并且对业务层透明。
	热点数据与非热点数据访问方案
访问特频繁容易出现单点瓶颈（比如业务只想部署一个节点，并且需要这个节点访问redis的性能很高）的业务（比如p3的大场景数据）直接访问redis，不走redis代理。
非热点数据可走redis代理。
2、	修改的数据自动从redis中同步到mongo中，数据半年（根据业务需要可配）没人访问则自动从redis中删除,删除的数据在redis中做个标签。
3、	Dbserver在redis中访问不到数据(并且知道数据再mongo中)则去mongo中拉取数据放redis中。
	因为数据经过半年（很长时间）才会淘汰出redis所以mongo有足够的时间同步数据，从mongo中取的数据必定是很久之前就不会修改的数据。所以dbsvr只需要连从mongo就可以取到最终的数据。
	可以通过增加从mongo的方式提高冷数据的服务能力，并且各个从mongo如果出问题只会对连接本mongo的业务有影响。可以做很好的故障隔离。
	因为冷数据在redis中打了标签，当在redis中访问不到数据时可以判断出此数据是冷数据还是库中不存在的数据，避免缓存穿透现象。
访问冷数据是在dbserver侧，并且dbserver侧是多线程的，所以访问冷数据和访问热数据的业务之间相互不会影响。

三、	需要开发的地方
1、	redis
	自动同步redis数据到mongo
	Redis中冷数据自动淘汰机制。
	Redis中需要将冷数据打标签（数据淘汰出redis，但是redis要知道哪些哪些数据是redis中没有但是mongo中可以访问到的）
	修改redis命令功能保证对redis操作的原子性（比如两个线程同时从mongo中拉同一数据到redis中，保证后面一个线程不覆盖前面一个线程的数据）
	Redis客户端驱动（从redis中取数据取不到并且判断出是冷数据则自动从mongo中拉取）做到操作mongo对业务透明。
	目前redis不支持精确的回退到任意时间点（秒级、至少是分钟级）。需要对落地回退方式做二次开发。
2、	mongo（可选）
	开发mongo网关。对mongo的访问做好控制与限流。
	Mongo备份方式目前采用导出数据的方式，耗时并且在备份的时候对线上mongo有一定影响。可探索其它备份方式（如直接拷贝mongo磁盘文件）。
四、	对业务的影响
1、	存储层从全redis方案切换到此方案需要考虑的风险
	如果访问到冷数据比较多的业务(例如一年没登陆的工作室账号突然集中上线)，访问数据的速度会大大降低，业务会超时或者访问失败。一般全redis方案不需要考虑这个情况，这个是采用此方案思维上需要转变的地方。
2、其它未知影响。
