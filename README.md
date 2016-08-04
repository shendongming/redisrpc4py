# redisrpc4py
redisrpc for python
RedisJsonRpc
-----------------
#基于redis的json rpc


##keys 的约定
1. 命名空间方便 隔离 服务(共享redis的开发/测试/开发之间的隔离)
   /api/ns 这个是命名空间 set 列表 永久存储
        ["xxxddfsdf","sdfsdfsdf","ggsdfsdfsd"]
        
2. 服务列表 /api/{ns}/services 
   这个是 set 实现的 永久存储
   例如 [ user , article ]
 
3. 服务接口列表
    例如:
    /api/{ns}/desc/user 这是一个字典 hash
        {
            "add_user":"{添加用户接口 json 格式 描述这个接口}" ,
            "login":"{登录接口,json 格式 描述接口}"
        }
    /api/{ns}/desc/article 文章模块
       {
            "pub_article":"发布文章接口",
            "get_list":"获取文章列表"
       }
    某个接口描述
    {
        "doc":"接口的文档描述",
        "args":[ "uid","article","接口参数描述"],
        "return":"返回类型"
    }
4. 响应key
   /api/{ns}/response/{request_id}


5. 请求 协议
    
    客户端请求服务
    lpush /api/{ns}/request/user  
    {
        "id":"{request uuid}"
        "version":"2",
        "method":"add_user",
        "params":["sdm","password"]
    }
    
    如果是通知 notification 这里和 json-rpc2.0 不兼容性,
     id 还有一个作用是请求的唯一标识
     {
        "id":"{request uuid}",
        "notification":true,
        "version":"2",
        "method":"add_user",
        "params":["sdm","password"]
    }
    如果是 notification rpc 请求,/api/{ns}/response/{request uuid} 结果将无法获取
    
    服务器 等待服务
    brpop /api/{ns}/request/user
    
    服务器执行 结果 推送到
    lpush /api/{ns}/response/{request uuid}
    {
        "id":"{request uuid}",
        "result":"{new uid}"
    }
    
    客户端 获取结果
    brpop /api/{ns}/response/{request uuid}
    获取结果 rpc 结束
    
    服务器可以启动多个进程/线程,监听请求

6. 高级请求

    1) 轮询模式,比较漫长的服务,客户端需要立刻响应.
        场景:浏览器发起一个请求,然后在异步定时轮询,
            可以使用长轮询(消耗并发连接资源),
            也可以使用短轮询(增加请求建立链接的资源)
            实现过程
            lpush /api/{ns}/request/user  {json rpc}
            长轮询:
            brpop /api/{ns}/response/{request id} 3 等待3s 获取结果
            短轮询
            rpop /api/{ns}/response/{request id}  获取结果,没有结果返回 null
    2) rpc 进度跟踪,消息通知
        lpush /api/request/user  {json rpc , 扩展字段 'track_process':1 }
        获取获取了 有了 跟踪进度的标记,那么服务端在执行任务的过程中 可以提交进度
        提交进度 1.1% 进度
        lpush /api/{ns}/process/{request id} "{time:时间戳,percent:0.011,message:消息内容}"  json内容传输
        EXPIRE 120
        设置 56.2% 进度
        lpush /api/process/{request id} "{time:时间戳,percent:0.562,message:消息内容}"
        EXPIRE 120 2分钟后过期,每人读取的进度,会被丢弃
         
7.works 工作节点列表
    /api/{ns}/works/ 服务工作节点列表
        /api/{ns}/works/user:node1 (下线删除节点)
        /api/{ns}/works/user:node2
        
8.基于session的 redis rpc (状态跟踪)
   1)group 组的概念 每组服务有一组节点组成
    /api/{ns}/groups/{group1}:{node1} 上线设置,下线清除
    设置ttl 定时刷新
   2)查询节点列表 
    keys /api/{ns}/groups/{group1}/*
   3)会话保持,会话映射, 定时刷新,过期消失
    先查表,没有在随机选择一个 session_id = group_id + ':' + client_id
    /api/{ns}/sess_map/{session_id1}= {group_id}:{node_id}
   4)等待服务请求
    brpop /api/{$ns}/sess_req/{group_id}:{node_id}/user    
   5)响应应答 response(没区别)             
    
    
6.监控 服务
    监控请求排队长度
        llen /api/request/user
        llen /api/request/article
    
    client list 获取客户端数量
    
7. 调用接口
    
---


Api 设计
------
底层 api

Class Service(object):
    def __init__(self, ns):
        pass
        


其他
work工作节点列表  这些服务有哪些工作节点 (这个职责 可以使用etcd 接管)
   例如  [ work1 , work2 ]
      
