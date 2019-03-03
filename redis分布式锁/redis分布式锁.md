```
public boolean lock(String key, Long value) {
    if (stringRedisTemplate.opsForValue().setIfAbsent(key, value.toString())) {
        return true;
    }
    //获取key的值，判断是是否超时
    String curVal = stringRedisTemplate.opsForValue().get(key);
    if (!StringUtils.isEmpty(curVal) && Long.parseLong(curVal) < System.currentTimeMillis()) {
        //获得之前的key值，同时设置当前的传入的value。这个地方可能几个线程同时过来，但是redis本身天然是单线程的，所以getAndSet方法还是会安全执行，
        //首先执行的线程，此时curVal当然和oldVal值相等，因为就是同一个值，之后该线程set了自己的value，后面的线程就取不到锁了
        String oldVal = stringRedisTemplate.opsForValue().getAndSet(key, value.toString());
        if(!StringUtils.isEmpty(oldVal) && oldVal.equals(curVal)) {
            return true;
        }
    }
    return false;
}
```