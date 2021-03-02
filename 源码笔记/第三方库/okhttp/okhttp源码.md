# 使用
## 同步
<pre>
Response response = client.newCall(request).execute();
</pre>
## 异步
<pre>
Request request = new Request.Builder()
    .url("http://publicobject.com/helloworld.txt")
    .build();

client.newCall(request).enqueue(new Callback() {
    @Override 
    public void onFailure(Call call, IOException e) {
      e.printStackTrace();
    }

    @Override 
    public void onResponse(Call call, Response response) throws IOException {
        ...
    }

</pre>

##   Response response = getResponseWithInterceptorChain();
- 不管是同步还是异步都会调用getResponseWithInterceptorChain 也就是拦截器链
# 重要类解释
## Dispatcher类：

- Dispatcher 通过维护一个线程池，来维护、管理、执行OKHttp的请求。
- Dispatcher 内部维护着三个队列：同步请求队列 runningSyncCalls、异步请求队列 runningAsyncCalls、异步缓存队列 readyAsyncCalls，和一个线程池 executorService。
- Dispatcher类整体可以参照生产者消费者模式来理解：
Dispatcher是生产者，executorService是消费者池，runningSyncCalls、runningAsyncCalls和readyAsyncCalls是消费者，用来消费请求Call。

## Interceptor
- 拦截器是Okhttp中提供的一种强大机制，它可以实现网络监听、请求、以及响应重写、请求失败重试等功能。
RetryAndFollowUpInterceptor、BridgeIntercept、CacheIntercept、ConnectIntercept、CallServerIntercept

# 拦截器链
## 内置拦截器
- RetryAndFollowUpInterceptor：负责失败重试以及重定向。
- BridgeInterceptor：负责把用户构造的请求转换为发送给服务器的请求，把服务器返回的响应转换为对用户友好的响应。
- CacheInterceptor：负责读取缓存以及更新缓存。
- ConnectInterceptor：负责与服务器建立连接。
- CallServerInterceptor：负责从服务器读取响应的数据。

## CacheInterceptor
- 缓存拦截器，缓存拦截器会根据请求的信息和缓存的响应的信息来判断是否存在缓存可用，如果有可以使用的缓存，那么就返回该缓存给用户，否则就继续使用责任链模式来从服务器中获取响应。当获取到响应的时候，又会把响应缓存到磁盘上面。
<pre>
Response intercept(Chain chain) throws IOException {
    //如果配置了缓存：优先从缓存中读取Response
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    long now = System.currentTimeMillis();
    //缓存策略，该策略通过某种规则来判断缓存是否有效
   CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    。。。。
    //如果根据缓存策略strategy禁止使用网络，并且缓存无效，直接返回空的Response
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          。。。
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)//空的body
          。。。
          .build();
    }

    //如果根据缓存策略strategy禁止使用网络，且有缓存则直接使用缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    //需要网络
    Response networkResponse = null;
    try {//执行下一个拦截器，发起网路请求
      networkResponse = chain.proceed(networkRequest);
    } finally {
      。。。
    }

    //本地有缓存，
    if (cacheResponse != null) {
      //并且服务器返回304状态码（说明缓存还没过期或服务器资源没修改）
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        //使用缓存数据
        Response response = cacheResponse.newBuilder()
            。。。
            .build();
          。。。。
         //返回缓存 
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    //如果网络资源已经修改：使用网络响应返回的最新数据
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    //将最新的数据缓存起来
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {

        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }
      。。。。
   //返回最新的数据
    return response;
  }
</pre>
### Cache类
- diskLruCache 实现

# 连接池
## 为什么需要连接池？
- 频繁的进行建立Sokcet连接和断开Socket是非常消耗网络资源和浪费时间的，所以HTTP中的keepalive连接对于降低延迟和提升速度有非常重要的作用。keepalive机制是什么呢？也就是可以在一次TCP连接中可以持续发送多份数据而不会断开连接。所以连接的多次使用，也就是复用就变得格外重要了，而复用连接就需要对连接进行管理，于是就有了连接池的概念。

- OkHttp中使用ConectionPool实现连接池，默认支持5个并发KeepAlive，默认链路生命为5分钟。

## 怎么实现的？
- 1）首先，ConectionPool中维护了一个双端队列Deque，也就是两端都可以进出的队列，用来存储连接。
- 2）然后在ConnectInterceptor，也就是负责建立连接的拦截器中，首先会找可用连接，也就是从连接池中去获取连接，具体的就是会调用到ConectionPool的get方法。

<pre>
RealConnection get(Address address, StreamAllocation streamAllocation, Route route) {
    assert (Thread.holdsLock(this));
    for (RealConnection connection : connections) {
      if (connection.isEligible(address, route)) {
        streamAllocation.acquire(connection, true);
        return connection;
      }
    }
    return null;
  }
</pre>
- 也就是遍历了双端队列，如果连接有效，就会调用acquire方法计数并返回这个连接。

- 3）如果没找到可用连接，就会创建新连接，并会把这个建立的连接加入到双端队列中，同时开始运行线程池中的线程，其实就是调用了ConectionPool的put方法。
<pre>
public final class ConnectionPool {
    void put(RealConnection connection) {
        if (!cleanupRunning) {
            //没有连接的时候调用
            cleanupRunning = true;
            executor.execute(cleanupRunnable);
        }
        connections.add(connection);
    }
}
</pre>
- 3）其实这个线程池中只有一个线程，是用来清理连接的，也就是上述的cleanupRunnable
<pre>
private final Runnable cleanupRunnable = new Runnable() {
        @Override
        public void run() {
            while (true) {
                //执行清理，并返回下次需要清理的时间。
                long waitNanos = cleanup(System.nanoTime());
                if (waitNanos == -1) return;
                if (waitNanos > 0) {
                    long waitMillis = waitNanos / 1000000L;
                    waitNanos -= (waitMillis * 1000000L);
                    synchronized (ConnectionPool.this) {
                        //在timeout时间内释放锁
                        try {
                            ConnectionPool.this.wait(waitMillis, (int) waitNanos);
                        } catch (InterruptedException ignored) {
                        }
                    }
                }
            }
        }
    };
</pre>
- 这个runnable会不停的调用cleanup方法清理线程池，并返回下一次清理的时间间隔，然后进入wait等待。

- 怎么清理的呢？看看源码：
<pre>
long cleanup(long now) {
    synchronized (this) {
      //遍历连接
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        //检查连接是否是空闲状态，
        //不是，则inUseConnectionCount + 1
        //是 ，则idleConnectionCount + 1
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }

        idleConnectionCount++;

        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }

      //如果超过keepAliveDurationNs或maxIdleConnections，
      //从双端队列connections中移除
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {      
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {      //如果空闲连接次数>0,返回将要到期的时间
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // 连接依然在使用中，返回保持连接的周期5分钟
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
</pre>
- 也就是当如果空闲连接maxIdleConnections超过5个或者keepalive时间大于5分钟，则将该连接清理掉。

- 4）这里有个问题，怎样属于空闲连接？
其实就是有关刚才说到的一个方法acquire计数方法：
<pre>
  public void acquire(RealConnection connection, boolean reportedAcquired) {
    assert (Thread.holdsLock(connectionPool));
    if (this.connection != null) throw new IllegalStateException();

    this.connection = connection;
    this.reportedAcquired = reportedAcquired;
    connection.allocations.add(new StreamAllocationReference(this, callStackTrace));
  }
</pre>
- 在RealConnection中，有一个StreamAllocation虚引用列表allocations。每创建一个连接，就会把连接对应的StreamAllocationReference添加进该列表中，如果连接关闭以后就将该对象移除。

- 5）连接池的工作就这么多，并不复杂，主要就是管理双端队列Deque<RealConnection>，可以用的连接就直接用，然后定期清理连接，同时通过对StreamAllocation的引用计数实现自动回收。