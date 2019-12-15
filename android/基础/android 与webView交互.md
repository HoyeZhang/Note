# js 调用 Android 方法

-  定义一个统一的类响应调用的动作  
<pre>
 */
public class JavascriptInterfaceClass {
    private Context context;                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            

    public JavascriptInterfaceClass (Context context) {
        this.context = context;
    }

    /**@JavascriptInterface 注解
     * 前端代码嵌入js：
     * 方法 名应和js函数方法名一致
     *
     */
    @JavascriptInterface
    public void callFunction(String src) {
        Log.e("src", src);
    }
}
</pre>

-  webview加载响应类 注意injectedObject 需一致
<pre>
webView.addJavascriptInterface(new JavascriptInterfaceClass(this), "injectedObject");
</pre>
# android 调用js方法
 - 4.4 以下 调用js方法，及调用并传参 无法直接获取返回值，可通过js再次调用Android获取返回值
	<pre>
mWebView.loadUrl("javascript:test('aa')");
</pre>
 - 4.4 以上 可接受返回值
 <pre>
 webView.evaluateJavascript("javascript:callJS()", new ValueCallback<String>() {
                            @Override
                            public void onReceiveValue(String s) {
                               Log.e("src", s);
                            }
                        });
</pre>