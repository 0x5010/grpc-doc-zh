gRPC连接backoff协议
================================

当我们连接到一个失败的后端时，通常希望不要立即重试(以避免泛滥的网络或服务器的请求)，而是做某种形式的指数backoff。

我们有几个参数：
1. INITIAL_BACKOFF (第一次失败重试前后需等待多久)
1. MULTIPLIER (在失败的重试后乘以的倍数)
1. JITTER (随机抖动因子).
1. MAX_BACKOFF (backoff上限)
1. MIN_CONNECT_TIMEOUT (最短重试间隔)

## 建议backoff算法
以指数形式返回连接尝试的起始时间，达到MAX_BACKOFF的极限，并带有抖动。
```vim
ConnectWithBackoff()
  current_backoff = INITIAL_BACKOFF
  current_deadline = now() + INITIAL_BACKOFF
  while (TryConnect(Max(current_deadline, now() + MIN_CONNECT_TIMEOUT))!= SUCCESS)
    SleepUntil(current_deadline)
    current_backoff = Min(current_backoff * MULTIPLIER, MAX_BACKOFF)
    current_deadline = now() + current_backoff + UniformRandom(-JITTER * current_backoff, JITTER * current_backoff)
```
参数默认值`MIN_CONNECT_TIMEOUT`=20sec `INITIAL_BACKOFF`=1sec `MULTIPLIER`=1.6 `MAX_BACKOFF`=120sec `JITTER`=0.2

根据的确切的关注点实现(例如最小化手机的唤醒次数)可能希望使用不同的算法，特别是不同的抖动逻辑。

备用的实现必须确保连接退避在同一时间开始分散，并且不得比上述算法更频繁地尝试连接。

## 重置backoff
backoff应在某个时间点重置为`INITIAL_BACKOFF`，以便重新连接行为是一致的，不管连接的是新开始的还是先前断开的连接。

当接收到`SETTINGS`帧时重置backoff，在那个时候，我们确定这个连接被服务器已经接受了。


