https://mp.weixin.qq.com/s/JqIJupVKUNuQYIDDxRtfqA

allocsvr:
1. 内存中储存最近一个分配出去的sequence：cur_seq，以及分配上限：max_seq
2. 分配sequence时，将cur_seq++，同时与分配上限max_seq比较：如果cur_seq > max_seq，将分配上限提升一个步长max_seq += step，并持久化max_seq
3. 重启时，读出持久化的max_seq，赋值给cur_seq

租约：
1. 租约失效：AllocSvr N秒内无法从StoreSvr读取加载配置时，AllocSvr停止服务
2. 租约生效：AllocSvr读取到新的加载配置后，立即卸载需要卸载的号段，需要加载的新号段等待N秒后提供服务

> AllocSvrB严格保证在AllocSvrA停止服务后提供服务

seqsvr所有模块使用了统一的路由表，描述了uid号段到AllocSvr的全映射。这份路由表由仲裁服务根据AllocSvr的服务状态生成，写到StoreSvr中，由AllocSvr当作租约读出，最后在业务返回包里旁路给Client端。

这里通过在Client端内存缓存路由表以及路由版本号来解决，请求步骤如下：

1. Client根据本地共享内存缓存的路由表，选择对应的AllocSvr；如果路由表不存在，随机选择一台AllocSvr
2. 对选中的AllocSvr发起请求，请求带上本地路由表的版本号
3. AllocSvr收到请求，除了处理sequence逻辑外，判断Client带上版本号是否最新，如果是旧版则在响应包中附上最新的路由表
4. Client收到响应包，除了处理sequence逻辑外，判断响应包是否带有新路由表。如果有，更新本地路由表，并决策是否返回第1步重试

每个用户有一个 cur_seq，uid 一共有 2^32 个， 32G 的数据
每个用户属于一个 section，一个 section 包含 10w 个 uid
一个 section 中的所有用户共享一个 max_seq

引入一个仲裁服务，探测AllocSvr的服务状态，决定每个uid段由哪台AllocSvr加载

storesvr 存储层，保存 max_seq；保存每台 allocsvr 负责哪些 uid
allocsvr 缓存层，分配 sequence；与 storesvr 保持租约
仲裁服务，探测 allocsvr 的服务状态

按照 uid 分 set

一个 uid 只能请求一个 allocsvr

storesvr：
1. 通过 raft 保存数据
2. 提供接口：
    1. add_host 添加一台 allocserver，返回成功
    2. heartbeat 添加过的 allocserver 租约，带上路由表版本，返回最新的路由表
    3. get_max_seq 获取
    4. update_max_seq 存储

allocsvr:
提供接口返回最新seq 或者对应的新的路由表
