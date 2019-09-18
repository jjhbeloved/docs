# 数据库连接池

## 断线重连 reconnect

TODO

db session 保存在 connection pool, 而这个session在 db restart的时候会无效. 这时候需要reconnect 获取新的 session. 可以参考 druid 连接池的解决方案.(当返回一场信息为xxx的时候, 重建链接)
