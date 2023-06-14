# Redis的基本数据类型和使用场景
- String: 
  - 使用String来缓存对象的整个JSON： SET user:1 '{"name":"cxk", "birth":"1998-08-02"}'
  - 将key分离成类似 user:{id}:{attribute}来用作缓存：MSET user:1:name cxk user:1:birth 1998-08-02 user:2:name wyf user:2:birth 1990-11-06