Safe Java-JS WebView Bridge
===================
抛弃使用高风险的WebView addjavascriptInterface方法，利用onJsPrompt解析纯JSON字符串，来实现网页JS层反射调用Java方法，同时通过对js层调用函数及回调函数的包装，支持方法参数传入所有已知的类型，包括number、string、boolean、object、function。

## 如何开始
初始化Webview WebSettings时允许js脚本执行，同时使用你的注入类实例化一个**InjectedChromeClient**对象，然后关联到你的Webview实例。如demo中的例子（指定的注入类为HostJsScope）：

	WebView wv = new WebView(this);
	WebSettings ws = wv.getSettings();
	ws.setJavaScriptEnabled(true);
	wv.setWebChromeClient(new InjectedChromeClient(HostJsScope.class));
	wv.loadUrl("file:///android_asset/test.html");
        
## 方法的定义
需要注入到网页的方法，**必须在注入类中定义为public static且第一个参数接收Webview**，其他参数的类型可以是int、long、double、boolean、String、JSONObject、JsCallback。方法执行时会默认将当前Webview的实例放到第一个参数，所以你的定义可能看起来像这样子的：
	
	public static void testA (WebView webView) {}
网页调用如下：
	
	HostApp.testA();

## 方法的重载
在定义时，支持不同参数类型或参数个数的方法重载，如：
	  
    public static int overloadMethod(WebView view, int val) {
        return val;
    }
    public static String overloadMethod(WebView view, String val) {
        return val;
    }
但需要注意的是，由于JS中数字类型不区分整型、长整型、浮点类型等，是统一由64位浮点数表示，故Java方法在定义时int/long/double被当作是一种类型，也即：
    
    public static int overloadMethod(WebView view, int val) {
        return val;
    }
    public static long overloadMethod(WebView view, long val) {
        return val;
    }
    public static double overloadMethod(WebView view, double val) {
        return val;
    }
上面这三个方法并没有发生重载，HostApp.overloadMethod(1)调用时只会调用最后一个定义的方法（double类型定义的那个）。

## 方法的返回值
Java层方法可以返回void 或 能转为字符串的类型（如int、long、String、double、float等）或 **可序列化的自定义类型**。关于自定义类型的返回可以参见Demo下“从Java层返回Java对象”项对HostApp.retJavaObject()的调用。另外如果方法定义时返回void，那么网页端调用得到的返回值为null。

如果方法执行过程中出现异常，那么在网页JS端会抛出异常，可以catch后打印详细的错误说明。

## 关于异步回调
举例说明，首先你可以在Java层定义如下方法，该方法的作用是延迟设定的时间之后，用你传入的参数回调Js函数：
  
    public static void delayJsCallBack(WebView view, int ms, final String backMsg, final JsCallback jsCallback) {
      TaskExecutor.scheduleTaskOnUiThread(ms*1000, new Runnable() {
          @Override
          public void run() {
              jsCallback.apply(backMsg);
          }
      });
    }
那么在网页端的调用如下：
  
    HostApp.delayJsCallBack(3, 'call back haha', function (msg) {
      HostApp.alert(msg);
    });
即3秒之后会弹出你传入的'call back haha'信息。
故从上面的例子我们可以看出，你在网页端定义的回调函数是可以附加多个参数，Java方法在执行回调时需要带入相应的实参就行了。当然这里的**回调函数的参数类型目前还不支持过复杂的类型，仅支持能够被转为字符串的类型**。

另外需要注意的是一般传入到Java方法的js function是一次性使用的，即在Java层jsCallback.apply(...)之后不能再发起回调了。如果需要传入的function能够在当前页面生命周期内多次使用，请在第一次apply前**setPermanent(true)**。例如：

	public static void setOnScrollBottomListener (WebView view, JsCallback jsCallback) {
		jsCallback.setPermanent(true);
		...
	}

## 小心过大数字
JS中使用过大数字时，可能会导致精度丢失或者错误的数字结果，如下面：

	HostApp.passLongType(14102300951321235)
传入一个大数**14102300951321235**到Java层，但Java层接收的数字实际上将会是**14102300951321236**这样一个错误的数字，所以当需要使用大数的情景下时，Java方法参数类型最好定义为**String类型**，而js层调用时也转为string，比如上面就为

	HostApp.passLongType(14102300951321235+'')。	

## Demo运行示意图
![image](https://github.com/pedant/safe-java-js-webview-bridge/raw/master/app-sample-screenshot.png)

更多实现细节见: http://www.pedant.cn/2014/07/04/webview-js-java-interface-research/
