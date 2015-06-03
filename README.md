# some technology blogs in this repo

httpClient使用总结

本次使用版本为：httpClient-4.2.5

HttpClient的两种使用方式——简单连接管理的HttpClient（BasicClientConnectionManager）和池化的HttpClient（PoolingClientConnectionManager）。

HttpClient从4.2开始抛弃了先前的SingleClientConnManager和ThreadSafeConnManger，取而代之的是BasicClientConnectionManager和PoolingClientConnectionManager。
BasicClientConnectionManager内部只维护一个活动的connection，尽管这个类是线程安全的，但是最好在一个单独的线程中重复使用它。如果在同一个BasicClientConnectionManager对象中，多次执行http请求，后继请求与先前请求是同一个route，那么BasicClientConnectionManager会使用同一个连接完成后续请求，否则，BasicClientConnectionManager会将先前的connection关闭，然后为后续请求创建一个新的连接。换句话说，BasicClientConnectionManager会尽力复用先前的连接（注意：创建连接和销毁连接都是不小的开销）
``` ``
注：这里的route是指远程访问的地址，如：www.taobao.com
``` 

Demo 1：
``` 

BasicClientConnectionManager：

    public static void basicClientTest() throws ClientProtocolException, IOException{
       HttpClient httpClient = new DefaultHttpClient();
       HttpGet httpGet = new HttpGet("http://m.weather.com.cn/data/101010100.html");
       HttpResponse response = httpClient.execute(httpGet);
       String result = EntityUtils.toString(response.getEntity(), Charset.forName("utf-8"));
       System.out.println(result);
       httpClient.getConnectionManager().shutdown();
    }
 

```

PoolingClientConnectionManager可以在多线程中使用，连接按照route被缓存（pooled），当后续的请求route已经在pool中存在，就会使用pool中先前使用的connection获取请求结果。PoolingClientConnectionManager对每个router维护的connection数目有上限要求，默认情况下，每个router最多维护两个并发线程的connection连接，整个pool最多容纳20个并发的connections。当然可以通过设置来修改这些限制。

Demo2：
```
PoolingClientConnectionManager：

    public static void httpclientPool() throws ClientProtocolException, IOException{
       SchemeRegistry registry = new SchemeRegistry();//创建schema
      
       SSLContext sslContext = null;//https类型的消息访问
       try{
           sslContext = SSLContext.getInstance("SSL");
           sslContext.init(null, null, null);
       }
       catch (Exception e) {
           e.printStackTrace();
       }
       SSLSocketFactory sslFactory = newSSLSocketFactory(sslContext,SSLSocketFactory.STRICT_HOSTNAME_VERIFIER);
       registry.register(new Scheme("http", 80, PlainSocketFactory.getSocketFactory()));//http 80 端口
       registry.register(new Scheme("https", 443, sslFactory));//https 443端口
      
       PoolingClientConnectionManager cm = newPoolingClientConnectionManager(registry);//创建connectionManager
      
       cm.setDefaultMaxPerRoute(20);//对每个指定连接的服务器（指定的ip）可以创建并发20 socket进行访问
       cm.setMaxTotal(200);//创建socket的上线是200
       HttpHost localhost = new HttpHost("locahost", 80);
       cm.setMaxPerRoute(new HttpRoute(localhost), 80);//对本机80端口的socket连接上限是80
      
       HttpClient httpClient = new DefaultHttpClient(cm);//使用连接池创建连接
       HttpParams params = httpClient.getParams();
       HttpConnectionParams.setSoTimeout(params, 60*1000);//设定连接等待时间
       HttpConnectionParams.setConnectionTimeout(params, 60*1000);//设定超时时间
 
       try{
           HttpGet httpGet = new HttpGet("http://m.weather.com.cn/data/101010100.html");
           HttpResponse response = httpClient.execute(httpGet);
           String result = EntityUtils.toString(response.getEntity(), Charset.forName("utf-8"));
           System.out.println(result);
       }finally{
           httpClient.getConnectionManager().shutdown();//用完了释放连接
       }
      
    }

``` 
项目中使用http请求维他命平台获取差异化车险定价因子等数据，开始时初始化的defaultHttpClient，如下所示：
``` 
    public static final HttpClient httpClient = new DefaultHttpClient();

```
上述httpClient使用了默认的初始化方法，即：使用的连接管理器是BasicClientConnectionManager。
处理单个请求时或者多个请求依次到达（第二个请求到达时，第一个请求已返回），一点问题没有，但是，处理并发请求时，问题就来了。。。

当多个请求同时到达，而此时httpClient只有一个有效的活动连接，如果前面的请求还没有返回，后面过来的请求就得等待，这样就会导致等待超时（获取连接失败）的问题

换用PoolingClientConnectionManager后问题解决，代码如下：
``` 
 public static final HttpClient httpClient = newSpecifiedHttpClient();

    /**
     * 获取定制版的httpClient实例
     * @return
     */
    private static HttpClient newSpecifiedHttpClient() {
        HttpParams params = new BasicHttpParams();
        //设置连接超时时间
        Integer CONNECTION_TIMEOUT = 5 * 1000; //设置请求超时5秒钟 根据业务调整
        Integer SO_TIMEOUT = 5 * 1000; //设置等待数据超时时间5秒钟 根据业务调整
        //定义了当从ClientConnectionManager中检索ManagedClientConnection实例时使用的毫秒级的超时时间
        //这个参数期望得到一个java.lang.Long类型的值。如果这个参数没有被设置，默认等于CONNECTION_TIMEOUT，因此一定要设置
        Long CONN_MANAGER_TIMEOUT = 500L; //该值就是连接不够用的时候等待超时时间，一定要设置，而且不能太大 ()

        params.setIntParameter(CoreConnectionPNames.CONNECTION_TIMEOUT, CONNECTION_TIMEOUT);
        params.setIntParameter(CoreConnectionPNames.SO_TIMEOUT, SO_TIMEOUT);
        params.setLongParameter(ClientPNames.CONN_MANAGER_TIMEOUT, CONN_MANAGER_TIMEOUT);
        //在提交请求之前 测试连接是否可用
        params.setBooleanParameter(CoreConnectionPNames.STALE_CONNECTION_CHECK, true);

        PoolingClientConnectionManager conMgr = new PoolingClientConnectionManager();
        conMgr.setMaxTotal(200); //设置整个连接池最大连接数 根据自己的场景决定
        //是路由的默认最大连接（该值默认为2），限制数量实际使用DefaultMaxPerRoute并非MaxTotal。
        //设置过小无法支持大并发(ConnectionPoolTimeoutException: Timeout waiting for connection from pool)，路由是对maxTotal的细分。
        conMgr.setDefaultMaxPerRoute(conMgr.getMaxTotal());//（目前只有一个路由-->维他命平台，因此让他等于最大值）
        return new DefaultHttpClient(conMgr, params);
    }

```

最后，请记得每个请求返回处理后，调用EntityUtils.toString或者EntityUtils.consume()关闭流
（如果不关闭流，请求量一多，你懂得。。。）

```
                responseContent = getHttpResult(responseEntity);
                //消费返回内容（关闭流）
                EntityUtils.consume(responseEntity);
```
