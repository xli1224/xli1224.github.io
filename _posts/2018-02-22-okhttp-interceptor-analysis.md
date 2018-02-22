---
layout: post
title: OKHttp 的 interceptor 实现
tags:
- Java
---

一直对于责任链模式具体如何实现没有一个很好的实践，最近刚好用 OKHttp mock 的时候发现它是用 Interceptor 实现的，而 Interceptor 的调用上看起来也是责任链模式的，可以钻进去看看。

OKHttp 可以为 client 设置 interceptor

```java
client = new OkHttpClient().newBuilder()
                .addInterceptor(mockInterceptor)
                .retryOnConnectionFailure(true).build();
```

Interceptor 的接口定义如下：

```java
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    /**
     * Returns the connection the request will be executed on. This is only available in the chains
     * of network interceptors; for application interceptors this is always null.
     */
    @Nullable Connection connection();

    Call call();

    int connectTimeoutMillis();

    Chain withConnectTimeout(int timeout, TimeUnit unit);

    int readTimeoutMillis();

    Chain withReadTimeout(int timeout, TimeUnit unit);

    int writeTimeoutMillis();

    Chain withWriteTimeout(int timeout, TimeUnit unit);
  }
}
```

Interceptor 接口本身只有一个 intercept 方法，它接受一个 Chain 对象。而 Chain 则是一个内部接口。

在使用 client 进行 http 请求时，OKHttpClient 先生成一个 RealCall 对象，请求发生在 RealCall 的 execute 方法里面。

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    eventListener.callStart(this);
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

上面对状态的修改和 listenter 的通知先不管， 看看 InterceptorChain 的调用 getResponseWithInterceptorChain。

```java
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }
```

初始化了一个 RealInterceptorChain，初始化的 index 是0。

```java
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
    RealConnection connection) throws IOException {
  if (index >= interceptors.size()) throw new AssertionError();

  calls++;

  // If we already have a stream, confirm that the incoming request will use it.
  if (this.httpCodec != null && !this.connection.supportsUrl(request.url())) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must retain the same host and port");
  }

  // If we already have a stream, confirm that this is the only call to chain.proceed().
  if (this.httpCodec != null && calls > 1) {
    throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
        + " must call proceed() exactly once");
  }

  // Call the next interceptor in the chain.
  RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
      connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
      writeTimeout);
  Interceptor interceptor = interceptors.get(index);
  Response response = interceptor.intercept(next);

  // Confirm that the next interceptor made its required call to chain.proceed().
  if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
    throw new IllegalStateException("network interceptor " + interceptor
        + " must call proceed() exactly once");
  }

  // Confirm that the intercepted response isn't null.
  if (response == null) {
    throw new NullPointerException("interceptor " + interceptor + " returned null");
  }

  if (response.body() == null) {
    throw new IllegalStateException(
        "interceptor " + interceptor + " returned a response with no body");
  }

  return response;
}
```

可以看到 RealInterceptorChain 的逻辑就是调用当前 index 的 inteceptor 的 intercept 方法。偷巧的地方在于，它构造了一个新的 RealInterceptorChain，将 index+1，交由当前的 Interceptor 决定是否要继续调用 chain 上面的 interceptor。Chain 本身不关心对于 chain 上面的所有 intercetpor 的调用或者返回值判断（null 除外），都交给 Interceptor 实现。

比如 Mock 的实现上，就是自定义了一套 Behavior，当 Behavior 是 relayed 时，并且当前没有任何匹配规则，MockInterceptor 就会将请求继续在 Chain 上传递。

```java
public Response intercept(Chain chain) throws IOException {
    Request request = chain.request();

    for (Iterator<Rule> it = rules.iterator(); it.hasNext(); ) {
        Rule rule = it.next();
        Response response = rule.accept(request);

        if (rule.isConsumed()) {
            it.remove();
        }
        if (response != null) {
            return response;

        } else if (behavior == Behavior.SEQUENTIAL) {
            StringBuilder sb = new StringBuilder("Not matched next rule: ");
            sb.append(rule);
            sb.append(", request=");
            sb.append(request);
            sb.append("\nFailed to match:");
            int i = 0;
            for (Map.Entry<Matcher, String> e : rule.getFailReason(request).entrySet()) {
                sb.append("\n\t");
                sb.append(++i);
                sb.append(": ");
                sb.append(e.getValue());
                sb.append("; matcher=");
                sb.append(e.getKey());
            }
            throw new AssertionError(sb.toString());
        }
    }

    // no matched rules or no more rules
    if (behavior == Behavior.RELAYED) {
        return chain.proceed(request);
    }

    StringBuilder sb = new StringBuilder("Not matched any rule: request=");
    sb.append(request);
    if (rules.isEmpty()) {
        sb.append("\nNo remaining rules!");

    } else {
        sb.append("\nRemaining rules:");
        int i = 0;
        for (Rule rule : rules) {
            sb.append("\n\t");
            sb.append(++i);
            sb.append(": ");
            sb.append(rule);
        }
    }
    throw new AssertionError(sb.toString());
}
```

OKHttp 还把真正的 Http 请求也做成了多个 Interceptor，也就是 RealCall 在构建 RealInterceptorChain 的时候会加进去的，包括：

-   RetryAndFollowUpInterceptor。它排在第一个，因为它要控制对于 chain 的重试。
-   BridgeInterceptor。负责将 UserRequest 转化为 NetworkRequest，添加各种 header。然后把 NetworkResponse 转化为 UserResponse，分析其中的 header 和 body 类型。
-   CacheInterceptor。这个就是本地 cache 了。可以缓存一些请求和返回值。
-   ConnectionInterceptor。创建连通目标服务器的 connection。
-   CallServerInterceptor。真正的 http 请求。

需要注意的是，自定义的 Interceptor 是排在这几个 Interceptor 前面的，这也给了自定义 Interceptor 去修改 request 和 response 的机会，只需要通过调用 chain.proceed()，就可以获取后续的 Interceptor 的返回值。

设计上的一些亮点：

1.  通过传递链条来让各组件自己控制是否继续。
2.  链条的调用带返回值，使得上游组件能够覆盖或者修改下游组件的入参和返回。
3.  链条的执行是自发的，由外部控制的，不是类似 manager 的统一控制模式，从而避免了复杂的返回值判断和传递。
4.  扩展性上，自定义的放在前面，固定的放在后面，这个算是比较常见的 default 实现了。不过还是那句话， chain 带返回值和控制移交给 Interceptor 是个很不错的设计，使得可交互性大大增强。
