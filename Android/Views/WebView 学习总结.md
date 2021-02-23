WebView是一个基于webkit引擎、展现web页面的控件。主要用于显示和渲染Web页面，直接使用html文件（网络上或本地assets中）作布局，可和JavaScript交互调用。

## 一、webView的一些方法

## 1.1 加载url

加载方式根据资源分为三种

    //1. 加载一个网页：
    webView.loadUrl("http://www.baidu.com");
    
    //2：加载apk包中的html页面
    webView.loadUrl("file:///android_asset/test.html");
    
    //3：加载手机本地的html页面
    webView.loadUrl("content://com.android.htmlfileprovider/sdcard/test.html");

## 1.2 WebView的状态

    //激活WebView为活跃状态，能正常执行网页的响应
    webView.onResume() ；
    
    //当页面被失去焦点被切换到后台不可见状态，需要执行onPause
    //通过onPause动作通知内核暂停所有的动作，比如DOM的解析、plugin的执行、JavaScript执行。
    webView.onPause()；
    
    //当应用程序(存在webview)被切换到后台时，这个方法不仅仅针对当前的webview而是全局的全应用程序的webview
    //它会暂停所有webview的layout，parsing，javascripttimer。降低CPU功耗。
    webView.pauseTimers()
    //恢复pauseTimers状态
    webView.resumeTimers()；
    
    //销毁Webview
    //在关闭了Activity时，如果Webview的音乐或视频，还在播放。就必须销毁Webview
    //但是注意：webview调用destory时,webview仍绑定在Activity上
    //这是由于自定义webview构建时传入了该Activity的context对象
    //因此需要先从父容器中移除webview,然后再销毁webview:
    rootLayout.removeView(webView); 
    webView.destroy();


## 1.3 前进后退
    Webview.canGoBack() 
    Webview.canGoForward()
    
    Webview.goBack()
    Webview.goForward()
    Webview.goBackOrForward(intsteps)

## 1.4 清除缓存：

    //清除网页访问留下的缓存
    //由于内核缓存是全局的因此这个方法不仅仅针对webview而是针对整个应用程序.
    Webview.clearCache(true);
    
    //清除当前webview访问的历史记录
    //只会webview访问历史记录里的所有记录除了当前访问记录
    Webview.clearHistory()；
    
    //这个api仅仅清除自动完成填充的表单数据，并不会清除WebView存储到本地的数据
    Webview.clearFormData()；

## 二、WebSettings

    WebSettings webSettings = webView.getSettings();
### 2.1 自适应屏幕
    webSettings.setUseWideViewPort(true);
    webSettings.setLoadWithOverviewMode(true);
### 2.2 缩放
    webSettings.setSupportZoom(true);
    webSettings.setBuiltInZoomControls(true);
    webSettings.setDisplayZoomControls(false);

### 2.3 缓存
    webSettings.setCacheMode(WebSettings.LOAD_DEFAULT);
    webSettings.setAppCachePath(getFilesDir().getAbsolutePath()+"mywebs");

### 2.4 缓存设置
    LOAD_CACHE_ONLY: 不使用网络，只读取本地缓存数据
    LOAD_DEFAULT: （默认）根据cache-control决定是否从网络上取数据。
    LOAD_NO_CACHE: 不使用缓存，只从网络获取数据.
    LOAD_CACHE_ELSE_NETWORK，只要本地有，无论是否过期，或者no-cache，都使用缓存中的数据。

### 2.5 其他
    webSettings.setAllowFileAccess(true);
    webSettings.setLoadsImagesAutomatically(true);
    webSettings.setDefaultTextEncodingName("utf-8");
    webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

## 三、WebViewClient
用于处理各种通知 & 请求事件
shouldOverrideUrlLoading()
onPageStarted()
onPageFinished()
onLoadResource()
onReceivedError()
onReceivedSslError()

## 四、WebChromeClient

辅助 WebView 处理 Javascript 的对话框,网站图标,网站标题等等。
onProgressChanged()
onReceivedTitle()
onJsAlert()
onJsConfirm()
onJsPrompt()

## 五、JS交互
### 1.Android 调用Js
对于Android调用JS代码的方法有2种：
1.1 通过WebView的loadUrl()
1.2 通过WebView的evaluateJavascript(), 使用时需进行版本判断(>=Android 4.4) 
    
    //需要先加载网页
    webView.loadUrl("file:///android_asset/javascript1.html");
    ...
    if (version < 18) {
        mWebView.loadUrl("javascript:callJS()");
    } else {
        mWebView.evaluateJavascript（"javascript:callJS()", new ValueCallback<String>() {
            @Override
            public void onReceiveValue(String value) {
                //此处为 js 返回的结果
            }
        });
    }

![pic1](https://leanote.com/api/file/getImage?fileId=59e0e569ab64412997001f6e)

特别注意：JS代码调用一定要在 onPageFinished（） 回调之后才能调用，否则不会调用


### 2.Js 调用Android
2.1 通过WebView的addJavascriptInterface()进行对象映射

2.2 在Android通过WebViewClient复写shouldOverrideUrlLoading()

2.3 通过 WebChromeClient 的onJsAlert()、onJsConfirm()、onJsPrompt（）方法回调拦截JS对话框alert()、confirm()、prompt() 消息

![pic2](https://leanote.com/api/file/getImage?fileId=59e0e86eab64412997001fb1)

## 六、内存泄漏

参考：
http://www.jianshu.com/p/3c94ae673e2a
http://www.jianshu.com/p/345f4d8a5cfa
http://www.jianshu.com/p/3a345d27cd42