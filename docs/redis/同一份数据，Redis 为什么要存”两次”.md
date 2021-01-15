









- sadd key member1 member2：将一个或多个元素 `member` 加入到集合 `key` 当中，并返回添加成功的数目，如果元素已存在则被忽略。
- sismember key member：判断元素 `member` 是否存在集合 `key` 中。
- srem key member1 member2：移除集合 `key` 中的元素，不存在的元素会被忽略。
- smove source dest member：将元素 `member` 从集合 `source` 中移动到 `dest` 中，如果 `member` 不存在，则不执行任何操作。
- smembers key：返回集合 `key` 中所有元素。