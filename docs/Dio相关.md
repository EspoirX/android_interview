### 通过Dio把请求转发到原生端
Dio 里面有个 HttpClientAdapter，默认实现貌似是 IOHttpClientAdapter ，通过自定义实现它，即可实现请求转发：
```dart
class NativeRequestClientAdapter implements HttpClientAdapter {
  //...
  @override
  Future<ResponseBody> fetch(
      RequestOptions options, Stream<Uint8List>? requestStream, Future<void>? cancelFuture) async {
    Map params = {};
    params["requestUrl"] = options.path;
    params["requestType"] = options.method;
    params["params"] = options.queryParameters;
    final Map nativeResponse = await FlutterHydra().invokeMethod('sendHttpRequest', params);
    String resultJson = nativeResponse["resultJson"] ?? "{}";
    int statusCode = nativeResponse["statusCode"] ?? "-1";
    String statusMessage = nativeResponse["statusMessage"] ?? "statusMessage is null";
    return ResponseBody.fromString(resultJson, statusCode, statusMessage: statusMessage);
  }
  //...
}
```
实现 HttpClientAdapter 主要是实现 fetch 方法，在 RequestOptions 里面可以拿到请求url，
请求类型（get,post），请求参数等等  
然后就是通过 **MethodChannel** 和原生沟通了，然后得到请求结果。  
fetch 需要返回一个 ResponseBody 对象，刚好它提供了一个 fromString 方法，所以通过 ResponseBody.fromString 包装
一下结果就行。

然后就可以正常使用了：
```dart
Future<Response> get(String url, Map<String, dynamic>? queryParameters) async {
    Response response = await _dio.get(url, queryParameters: queryParameters);
    return response;
}

Future<Response> post(String url, Map<String, dynamic>? queryParameters) async {
    Response response = await _dio.post(url, queryParameters: queryParameters);
    return response;
}
```
请求结果都在 Response 里面。

**ResponseBody注意点**  
值得注意的是 ResponseBody 的 statusCode 不能够乱传，在初始化 Dio 时，如果你没自定义 BaseOptions，
内部会默认 new 一个，  
然后 BaseOptions 继承 _RequestConfig，在 _RequestConfig 里面，对于参数 validateStatus 有个默认实现：
```dart
this.validateStatus = validateStatus ??
    (int? status) {
      return status != null && status >= 200 && status < 300;
    };
```
意思是如果你的 statusCode 不在这个范围内，会认为是请求失败，抛出 DioError 异常  
如果你的请求是 0 代表成功的话，就需要自己实现一下 validateStatus 了：
```dart
  _dio = Dio(BaseOptions(validateStatus: (int? status) {
      return status != null && status >= 0; //认为大于0就成功，否则会抛 DioError
    }));
```


**除了 get，post 这些基本请求，如果我要发起 RPC 请求怎么封装？**  
dio 的 post，get等请求最后都会调用 request 方法：
```dart
  Future<Response<T>> request<T>(
    String path, {
    Object? data,
    Map<String, dynamic>? queryParameters,
    CancelToken? cancelToken,
    Options? options,
    ProgressCallback? onSendProgress,
    ProgressCallback? onReceiveProgress,
  });
```
然后你会发现请求什么类型都是 Options 参数控制的，那么我们就可以自己搞一个：
```dart
  Future<Response<T>> _rpcRequest<T>(
    Map<String, dynamic>? queryParameters, {
    Object? data,
    Options? options,
    CancelToken? cancelToken,
    ProgressCallback? onReceiveProgress,
  }) {
    return _dio.request<T>(
      "",
      data: data,
      queryParameters: queryParameters,
      options: checkOptions('RPC', options),
      onReceiveProgress: onReceiveProgress,
      cancelToken: cancelToken,
    );
  }

  static Options checkOptions(String method, Options? options) {
    options ??= Options();
    options.method = method;
    return options;
  }
```

然后在 HttpClientAdapter 中，通过判断 options.method == "RPC" 去执行 RPC 相关操作就可以。    
随便一提，在和原生沟通的过程中 rpc 请求结果不能直接传对象，要传 byte（对于代码来说就是 dynamic 类型），
然后在接受后再组装成对象。

### Dio下载封装
找了个开源库 [flowder](https://github.com/Crdzbird/flowder)  ，感觉还行，然后自己修改了一下变得更好用，得到新的下载框架
[Flowder](https://github.com/EspoirX/flutterUseful/blob/main/lib/download/Flowder.dart)  
用法如下：
```dart
Flowder.download(
    "url",
    DownloadOptions(childDir: "img", fileName: "1.jpg")
      ..progress = (current, total) {
         //...
      }
      ..onSuccess = (path) {
        //...
      }
      ..onFail = (code, msg) {
        //...
      });
```
默认的下载路径是沙盒路径，保存路径是 沙盒/childDir/fileName  
所以只需要传入下载url，childDir 和 fileName 即可。如果要自定义路径，则传入参数 file，file随你定路径。  
下载多个文件可在 for 循环里面调用