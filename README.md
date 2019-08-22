retrofit-helper 

> Retrofit是很多android开发者都在使用的Http请求库！他负责网络请求接口的封装,底层实现是OkHttp,它的一个特点是包含了特别多注解，通过动态代理的方式使得开发者在使用访问网络的时候更加方便简单高效。



- #### 1. Retrofit-helper扩展了那些功能

  | 描述                                             | 相关类和方法                                                 |
  | ------------------------------------------------ | ------------------------------------------------------------ |
  | 回调函数中直接处理请求结果，无需再次判断是否成功 | `Callback.onSuccess(Call<T> call, T response)`               |
  | 请求开始和结束监听                               | `Callback.onStart(Call<T> call)`  和`Callback.onCompleted(Call<T> call, @Nullable Throwable t)` |
  | 全局维护多个Retrofit实例                         | `RetrofitFactory.DEFAULT` 和 `RetrofitFactory.OTHERS`        |
  | 统一解析异常信息                                 | `Callback.parseThrowable(Call<T> call, Throwable t)`         |
  | 绑定Activity或者Fragment生命周期                 | `LifeCall<T> bindToLifecycle(LifecycleProvider provider, Lifecycle.Event event)` |
  | 拦截器监听下载和上传进度                         | `ProgressInterceptor` 、`ProgressListener`                   |
  | 单独指定某个请求的日志级别                       | `HttpLoggingInterceptor`                                     |

- #### 2. 封装逻辑解析

  - 2.1  `RetrofitFactory`全局管理`retrofit`实例

     全局管理retrofit 实例，通用的做法有单利模式或者静态类成员属性的方式，并且在Application中初始化，在这里只是管理全局retrofit对象，故采用静态的成员属性即可满足需求

    DEFAULT 静态变量管理默认常用的的retrofit对象，OTHERS 管理其他多个不同配置的retrofit
    
    ```java
    /**
     * 创建时间：2018/4/3
     * 编写人： chengxin
     * 功能描述：管理全局的Retrofit实例
     */
    public final class RetrofitFactory {
        /**
         * 缓存不同配置的retrofit集合，如不同的url ,converter等
         */
        public static final Map<String, Retrofit> OTHERS = new ConcurrentHashMap<>(2);
        /**
         * 全局的Retrofit对象
         */
        public static volatile Retrofit DEFAULT;
        /**
         * A {@code null} value is permitted
         */
        @Nullable
        public static volatile OnEventListener LISTENER;
    
        private RetrofitFactory() {
        }
    
        public static <T> T create(Class<T> service) {
            //确保多线程的情况下retrofit不为空或者被修改了
            Retrofit retrofit = DEFAULT;
            Utils.checkState(retrofit != null, "DEFAULT == null");
            return retrofit.create(service);
        }
    
        /**
         * @param name 获取 OTHERS 中指定名字的retrofit
         */
        public static <T> T create(String name, Class<T> service) {
            Utils.checkNotNull(name, "name == null");
            Retrofit retrofit = OTHERS.get(name);
            Utils.checkState(retrofit != null,
                    String.format("retrofit named with '%s' was not found , have you put it in OTHERS ?", name));
            return retrofit.create(service);
        }
    }
    ```
    
  - 2.2  自定义Call,支持绑定生命周期。`Call`接口实现 ` enqueue(Callback<T> callback)`方法 `enqueue(Callback<T> callback)` ，支持绑定Activity或者Fragment生命周期`bindToLifecycle(LifecycleProvider provider, Lifecycle.Event event)`

    ```java
    /**
     * 创建时间：2018/4/8
     * 编写人： chengxin
     * 功能描述：支持生命周期绑定的Call{@link retrofit2.Call}
     */
    public interface Call<T> extends Callable<T>, Cloneable {
    
        String TAG = Call.class.getSimpleName();
    
        boolean isExecuted();
    
        void cancel();
    
        boolean isCanceled();
    
        Call<T> clone();
    
        Request request();
    
        /**
         * 绑定生命周期
         *
         * @param provider LifecycleProvider
         * @param event    {@link Lifecycle.Event}
         * @return
         */
        LifeCall<T> bindToLifecycle(LifecycleProvider provider, Lifecycle.Event event);
    
       /**
         * default event is {@link Lifecycle.Event#ON_DESTROY}
         *
         * @param provider LifecycleProvider
         * @return LifeCall
         * @see Call#bindToLifecycle(LifecycleProvider, Lifecycle.Event)
         */
        LifeCall<T> bindUntilDestroy(LifecycleProvider provider);
    }
    ```

    

  - 2.3  `Callback` 统一处理回调和异常

    retrofit 默认的callback只有支持成功和失败的回调，而一般我们的场景需要在请求开始和结束的地方做一些UI的展示处理，如显示加载动画等，故添加了`onStart` 和`onCompleted ` 监听。`parseThrowable`方法可以处理各种请求过程中抛出的throwable,转换成统一的格式方便我们处理展示等

    ```java
  /**
     * if {@link LifeCall#isDisposed()} return true, all methods will not call
     *
     * @param <T> Successful response body type.
     */
    @UiThread
    public interface Callback<T> {
        /**
         * @param call The {@code Call} that was started
         */
        void onStart(Call<T> call);
    
        /**
         * @param call The {@code Call} that has thrown exception
         * @param t    统一解析throwable对象转换为HttpError对象，如果throwable为{@link HttpError}
         *             <li>则为{@link retrofit2.Converter#convert(Object)}内抛出的异常</li>
         *             如果为{@link retrofit2.HttpException}
         *             <li>则为{@link Response#body()}为null的时候抛出的</li>
         */
        @NonNull
        HttpError parseThrowable(Call<T> call, Throwable t);
    
        /**
         * 过滤一次数据,如剔除List中的null等,默认可以返回t
         */
        @NonNull
        T transform(Call<T> call, T t);
    
        void onError(Call<T> call, HttpError error);
    
        void onSuccess(Call<T> call, T t);
    
        /**
         * @param t 请求失败的错误信息
         */
        void onCompleted(Call<T> call, @Nullable Throwable t);
    }
    ```
    
    
    
    ​		
    
  - 2.4  `HttpError` 统一包装异常错误，由Callback `parseThrowable` 方法统一返回

    ​       HttpError类中有两个成员属性msg 被body，msg是保存错误的描述信息等，body可以保存异常的具体信息或者原始的json等，`onError(Call<T> call, HttpError error)`回调方法可以根据body的具体信息做二次处理。

    ```java
    /**
     * 通用的错误信息，一般请求是失败只需要弹出一些错误信息即可,like{@link retrofit2.HttpException}
     * Created by chengxin on 2017/6/22.
     */
    public final class HttpError extends RuntimeException {
        private static final long serialVersionUID = -134024482758434333L;
        /**
         * 展示在前端的错误描述信息
         */
        public String msg;
    
        /**
         * <p>
         * 请求失败保存失败信息,for example:
         * <li>BusiModel: {code:xxx,msg:xxx} 业务错误信息</li>
         * <li>original json:  原始的json</li>
         * <li>{@link retrofit2.Response}:错误响应体->Response<?></li>
         * <li>Throwable: 抛出的异常信息</li>
         * </p>
         */
        @Nullable
        public final transient Object body;
    
        public HttpError(String msg) {
            this(msg, null);
        }
    
        public HttpError(String msg, @Nullable Object body) {
            super(msg);
            if (body instanceof Throwable) {
                initCause((Throwable) body);
            }
            //FastPrintWriter#print(String str)
            this.msg = msg != null ? msg : "null";
            this.body = body;
        }
    
        /**
         * 保证和msg一致
         */
        @Override
        public String getMessage() {
            return msg;
        }
    
        @Override
        public String toString() {
            return "HttpError {msg="
                    + msg
                    + ", body="
                    + body
                    + '}';
        }
    }
    ```

    

  - 2.5  CallAdapterFactory返回`Call`请求适配器

    处理请求接口方法返回为Call的请求适配器工厂类

    ```java
    public final class CallAdapterFactory extends CallAdapter.Factory {
        private static final String RETURN_TYPE = Call.class.getSimpleName();
    
        public static final CallAdapter.Factory INSTANCE = new CallAdapterFactory();
    
        private CallAdapterFactory() {
        }
    
        /**
         * Extract the raw class type from {@code type}. For example, the type representing
         * {@code List<? extends Runnable>} returns {@code List.class}.
         */
        public static Class<?> getRawType(Type type) {
            return CallAdapter.Factory.getRawType(type);
        }
    
        @Override
        public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
            if (getRawType(returnType) != Call.class) {
                return null;
            }
            if (!(returnType instanceof ParameterizedType)) {
                throw new IllegalArgumentException(
                        String.format("%s return type must be parameterized as %s<Foo> or %s<? extends Foo>", RETURN_TYPE, RETURN_TYPE, RETURN_TYPE));
            }
            final Type responseType = getParameterUpperBound(0, (ParameterizedType) returnType);
    
            return new CallAdapter<Object, Call<?>>() {
                @Override
                public Type responseType() {
                    return responseType;
                }
    
                @Override
                public Call<Object> adapt(retrofit2.Call<Object> call) {
                    return new RealCall<>(call);
                }
            };
        }
    }
    ```
    
  - 2.6  `RealLifeCall` 继承`LifeCall`实现Activity或Fragment生命周期绑定，自动管理生命周期

     负责生命周期的监听，在`onChanged(@NonNull Lifecycle.Event event)`方法中如果匹配到指定的event，标记为disposed 并取消请求，这时Callback中所有的回调函数将不会在执行，保证回调处的安全性。

    ```java
    
    final class RealLifeCall<T> implements LifeCall<T> {
        private final Call<T> delegate;
        private final Lifecycle.Event event;
        private final LifecycleProvider provider;
        /**
         * LifeCall是否被释放了
         * like rxAndroid MainThreadDisposable or rxJava ObservableUnsubscribeOn, IoScheduler
         */
        private final AtomicBoolean once = new AtomicBoolean();
    
        RealLifeCall(Call<T> delegate, Lifecycle.Event event, LifecycleProvider provider) {
            this.delegate = delegate;
            this.event = event;
            this.provider = provider;
            provider.observe(this);
        }
    
        @Override
        public void enqueue(final Callback<T> callback) {
            Utils.checkNotNull(callback, "callback==null");
            delegate.enqueue(new Callback<T>() {
                @Override
                public void onStart(Call<T> call) {
                    if (!isDisposed()) {
                        callback.onStart(call);
                    }
                }
    
                @NonNull
                @Override
                public HttpError parseThrowable(Call<T> call, Throwable t) {
                    if (!isDisposed()) {
                        return callback.parseThrowable(call, t);
                    }
                    return new HttpError("Already disposed.", t);
                }
    
                @NonNull
                @Override
                public T transform(Call<T> call, T t) {
                    if (!isDisposed()) {
                        return callback.transform(call, t);
                    }
                    return t;
                }
    
                @Override
                public void onSuccess(Call<T> call, T t) {
                    if (!isDisposed()) {
                        callback.onSuccess(call, t);
                    }
                }
    
                @Override
                public void onError(Call<T> call, HttpError error) {
                    if (!isDisposed()) {
                        callback.onError(call, error);
                    }
                }
    
                @Override
                public void onCompleted(Call<T> call, @Nullable Throwable t) {
                    if (!isDisposed()) {
                        callback.onCompleted(call, t);
                        provider.removeObserver(RealLifeCall.this);
                    }
                }
            });
        }
    
        @NonNull
        @Override
        public T execute() throws Throwable {
            try {
                if (isDisposed()) {
                    throw new DisposedException("Already disposed.");
                }
                T body = delegate.execute();
                if (isDisposed()) {
                    throw new DisposedException("Already disposed.");
                }
                return body;
            } catch (Throwable t) {
                if (isDisposed() && !(t instanceof DisposedException)) {
                    throw new DisposedException("Already disposed.", t);
                }
                throw t;
            } finally {
                if (!isDisposed()) {
                    provider.removeObserver(this);
                }
            }
        }
    
        @Override
        public void onChanged(@NonNull Lifecycle.Event event) {
            if (this.event == event
                    || event == Lifecycle.Event.ON_DESTROY
                    //Activity和Fragment的生命周期是不会传入 {@code Lifecycle.Event.ON_ANY},
                    //可以手动调用此方法传入 {@code Lifecycle.Event.ON_ANY},用于区分是否为手动调  用
                    || event == Lifecycle.Event.ON_ANY) {
                if (once.compareAndSet(false, true)/*保证原子性*/) {
                    delegate.cancel();
                    RetrofitFactory.getOnEventListener().onDisposed(delegate, event);
                    provider.removeObserver(this);
                }
            }
        }
    
        @Override
        public boolean isDisposed() {
            return once.get();
        }
    }
    ```
    
    
    
  - 2.7  `AndroidLifecycle`观察者模式统一分发生命周期事件

    继承LifecycleObserver 接口监听当前的Activity或者Fragment的生命周期，分发生命周期事件到Observer的`onChanged`方法。这个类是线程安全的，保证多线程环境的正确性

    ```java
    
    /**
     * 实现LifecycleObserver监听Activity和Fragment的生命周期
     * It is thread safe.
     *
     * @see android.database.Observable
     */
    public final class AndroidLifecycle implements LifecycleProvider, LifecycleObserver {
        private final Object mLock = new Object();
    
        @GuardedBy("mLock")
        private final ArrayList<Observer> mObservers = new ArrayList<>();
        /**
         * 缓存当前的Event事件
         */
        @GuardedBy("mLock")
        @Nullable
        private Lifecycle.Event mEvent;
    
        @MainThread
        public static LifecycleProvider createLifecycleProvider(LifecycleOwner owner) {
            return new AndroidLifecycle(owner);
        }
    
        private AndroidLifecycle(LifecycleOwner owner) {
            owner.getLifecycle().addObserver(this);
        }
    
        @OnLifecycleEvent(Lifecycle.Event.ON_ANY)
        void onEvent(LifecycleOwner owner, Lifecycle.Event event) {
            synchronized (mLock) {
                //保证线程的可见性
                mEvent = event;
                // since onChanged() is implemented by the app, it could do anything, including
                // removing itself from {@link mObservers} - and that could cause problems if
                // an iterator is used on the ArrayList {@link mObservers}.
                // to avoid such problems, just march thru the list in the reverse order.
                for (int i = mObservers.size() - 1; i >= 0; i--) {
                    mObservers.get(i).onChanged(event);
                }
            }
            if (event == Lifecycle.Event.ON_DESTROY) {
                owner.getLifecycle().removeObserver(this);
            }
        }
    
        @Override
        public void observe(Observer observer) {
            if (observer == null) {
                throw new IllegalArgumentException("The observer is null.");
            }
            synchronized (mLock) {
                if (mObservers.contains(observer)) {
                    return;
                }
                mObservers.add(observer);
                RetrofitFactory.getOnEventListener().onObserverCountChanged(this, mObservers.size() - 1, mObservers.size());
                // since onChanged() is implemented by the app, it could do anything, including
                // removing itself from {@link mObservers} - and that could cause problems if
                // an iterator is used on the ArrayList {@link mObservers}.
                // to avoid such problems, just call onChanged() after {@code mObservers.add(observer)}
                if (mEvent != null) {
                    observer.onChanged(mEvent);
                }
            }
        }
    
        @Override
        public void removeObserver(Observer observer) {
            if (observer == null) {
                throw new IllegalArgumentException("The observer is null.");
            }
            synchronized (mLock) {
                int index = mObservers.indexOf(observer);
                if (index == -1) {
                    return;
                }
                mObservers.remove(index);
                RetrofitFactory.getOnEventListener().onObserverCountChanged(this, mObservers.size() + 1, mObservers.size());
            }
        }
    
        @Override
        public String toString() {
            return "AndroidLifecycle@" + Integer.toHexString(hashCode());
        }
    }
    ```
    
    
    
  - 2.8  `ProgressInterceptor` 拦截器监听下载和上传进度

     继承`okhttp3.Interceptor`  ，构造方法中传入`ProgressListener`监听进度

    ```java
    /**
     * 创建时间：2018/8/2
     * 编写人： chengxin
     * 功能描述：上传或下载进度监听拦截器
     */
    public class ProgressInterceptor implements Interceptor {
    
        private final ProgressListener mProgressListener;
    
        public ProgressInterceptor(ProgressListener progressListener) {
            Utils.checkNotNull(progressListener, "progressListener==null");
            this.mProgressListener = progressListener;
        }
    
        @Override
        public Response intercept(Chain chain) throws IOException {
            Request request = chain.request();
            RequestBody requestBody = request.body();
            //判断是否有上传需求
            if (requestBody != null && requestBody.contentLength() > 0) {
                Request.Builder builder = request.newBuilder();
                RequestBody newRequestBody = new ProgressRequestBody(requestBody, mProgressListener, request);
                request = builder.method(request.method(), newRequestBody).build();
            }
    
            Response response = chain.proceed(request);
            ResponseBody responseBody = response.body();
            if (responseBody != null && responseBody.contentLength() > 0) {
                Response.Builder builder = response.newBuilder();
                ResponseBody newResponseBody = new ProgressResponseBody(responseBody, mProgressListener, request);
                response = builder.body(newResponseBody).build();
            }
            return response;
        }
    }
    ```

  - 2.9  `HttpLoggingInterceptor` 可以单独指定某个请求的日志级别

    构造OkhttpClient时添加此拦截器，在请求的服务方法中添加注解

    @Headers("LogLevel:NONE") 或 @Headers("LogLevel:BASIC") 或 @Headers("LogLevel:HEADERS") 或@Headers("LogLevel:BODY")

    ```java
    @FormUrlEncoded
    @Headers("LogLevel:HEADERS")
    @POST("user/login")
    Call2<LoginInfo> getLogin(@Field("username") String username, @Field("password") String password);
    ```

- #### 3.实战

  - 3.1 初始化全局Retrofit对象

    ```java
    Retrofit retrofit = new Retrofit.Builder()
                    .baseUrl("http://wanandroid.com/")
                    .callFactory(new OkHttpClient.Builder()
                            .addNetworkInterceptor(httpLoggingInterceptor)
                            .build())
                    //必须添加此adapter 用于构建Call
                    .addCallAdapterFactory(CallAdapterFactory.INSTANCE)
                    //添加自定义json解析器 
                    .addConverterFactory(GsonConverterFactory.create())
                    .build();
     RetrofitFactory.DEFAULT = retrofit;
     
     //可以添加多个，如：
     RetrofitFactory.OTHERS.put("other",otherRetrofit);
     
    ```

  - 3.2 添加请求服务接口

    下面为登录的 post请求

    ```java
    @FormUrlEncoded
    @POST("user/login")
    Call2<LoginInfo> getLogin(@Field("username") String username, @Field("password") String password);
    ```

  - 3.3 发起请求

    是不是很简单，妈妈再也不用担心Activity销毁后资源回收导致的NullPointException等问题了

  ```java
   
  public class MainActivity extends AppCompatActivity {
  
      LifecycleProvider provider = AndroidLifecycle.createLifecycleProvider(this);
  
      @Override
      protected void onCreate(@Nullable Bundle savedInstanceState) {
          super.onCreate(savedInstanceState);
          setContentView(R.layout.activity_main);
          final Context context = this;
          
          RetrofitFactory.create(ApiService.class)
                  .getLogin("loginName", "password")
                //.bindUntilDestroy(provider) Activity销毁时取消请求
                  .bindToLifecycle(provider, Lifecycle.Event.ON_STOP)
                .enqueue(new DefaultCallback<LoginInfo>() {
                      @Override
                    public void onStart(Call<LoginInfo> call) {
                          showLoading();
                    }
  
                      @Override
                      public void onError(Call<LoginInfo> call, HttpError error) {
                          Toast.makeText(context, error.msg, Toast.LENGTH_SHORT).show();
                      }
  
                      @Override
                      public void onSuccess(Call<LoginInfo> call, LoginInfo loginInfo) {
                          Toast.makeText(context, "登录成功！", Toast.LENGTH_SHORT).show();
                          //do...
                      }
  
                      @Override
                      public void onCompleted(Call<LoginInfo> call, @Nullable Throwable t){
                          hideLoading();
                      }
                  });
      }
  }
  ```
  
- #### 4.注意事项

  - 4.1 构建retrofit是需要CallAdapterFactory实例，否则无法处理返回为Call的服务接口

  - 4.2 `Callback`的回调函数均在主线程执行，如果Call绑定了生命周期触发了`cancel()`方法

    UI回调方法均不会执行，如果要监听那些请求被取消了，可以设置`RetrofitFactory.LISTENER`属性，其为一个全局的监听器`OnEventListener`。
    
    

- #### 5.下载

  ```groovy
  dependencies {
       implementation 'com.xcheng:retrofit-helper:1.5.5'
  }
  ```
  

​        github地址: [retrofit-helper](https://github.com/xchengDroid/retrofit-helper)

  

#### License

```
Copyright 2019 xchengDroid

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

