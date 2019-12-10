# android 6.0 （api 23）
    

----------
> android 6.0相对于之前版本主要是加入了权限申请的功能需要适配


## 权限申请步骤
- 检查是否有权限操作 
<pre>
ContextCompat.checkSelfPermission(Context context, String permission);
 返回值 PackageManager.PERMISSION_GRANTED （有权限） PackageManager.PERMISSION_DENIED （无权限）  
</pre>

- 申请权限
<pre>
ActivityCompat.requestPermissions(Activity activity,String[] permissions, int requestCode);
</pre>
- 申请结果回调
<pre>
@Override
public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults)；

</pre>
## 以内存和拍照权限为例
- 申请
<pre>
private void initPermissions() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA) != PackageManager.PERMISSION_GRANTED
                || ContextCompat.checkSelfPermission(this, Manifest.permission.WRITE_EXTERNAL_STORAGE) != PackageManager.PERMISSION_GRANTED) {
            //申请权限
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.CAMERA, Manifest.permission.WRITE_EXTERNAL_STORAGE}, CAMERA_PRIMISSIONS_REQUEST_CODE);
        } else {
            if (hasSdcard()) {
                 //TODO 拍照
            } else {
                Toast.makeText(this, "无SD卡！", Toast.LENGTH_SHORT).show();
            }
        }
    }
</pre>
- 申请回调
<pre>
@Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        switch (requestCode) {
            case CAMERA_PRIMISSIONS_REQUEST_CODE:
                boolean agreeCamera = grantResults[0] == PackageManager.PERMISSION_GRANTED;
                boolean agreeStorage = grantResults[1] == PackageManager.PERMISSION_GRANTED;
                if (agreeCamera && agreeStorage) {
                    //两个权限都同意
                    //TODO 拍照

                } else {//有一个不同意
                    Toast.makeText(this, "您有权限未允许，无法拍照", Toast.LENGTH_SHORT).show();
                }
                break;
            default:
                break;
        }
    }
</pre>

# android 7.0 (api 24)

----------

> 需要去除项目中传递file://类似格式的uri。需要FileProvider 来处理 android.os.FileUriExposedException
## 在AndroidManifest中声明provider
<provider
    android:name="android.support.v4.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>

## 编写resource xml file
<pre>
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <root-path name="root" path="" />
    <files-path name="files" path="" />
    <cache-path name="cache" path="" />
    <external-path name="external" path="" />
    <external-files-path name="name" path="path" />
     <external-cache-path name="name" path="path" />
</paths>

</pre>

## 使用
<pre>
 Uri photoUri;
        if (android.os.Build.VERSION.SDK_INT < 24) {
            photoUri = Uri.fromFile(file);
        } else {
            photoUri = FileProvider.getUriForFile(this, getPackageName()+".fileprovider", file);
        }
</pre>

# android 8.0 (api 26)

----------

> 8.0手机奔溃  在android 8.0的手机上会遇到
* Only fullscreen activities can request orientation
8.0版本时为了支持全面屏，增加了一个限制：如果是透明的Activity，则不能固定它的方向，因为它的方向其实是依赖其父Activity的（因为透明）。然而这个bug只有在8.0中有，8.1中已经修复。具体crash有两种：

1.Activity的风格为透明，在manifest文件中指定了一个方向，则在onCreate中crash

2.Activity的风格为透明，如果调用setRequestedOrientation方法固定方向，则crash
## 解决代码
'''
 @Override
    protected void onCreate(Bundle savedInstanceState) {
        if (Build.VERSION.SDK_INT == Build.VERSION_CODES.O && isTranslucentOrFloating()) {
            boolean result = fixOrientation();
        }

        super.onCreate(savedInstanceState);
        // 设置为横屏模式
//        setRequestedOrientation(HuosdkInnerManager.getInstance().getScreenOrientation());
    }

    private boolean fixOrientation(){
        try {
            Field field = Activity.class.getDeclaredField("mActivityInfo");
            field.setAccessible(true);
            ActivityInfo o = (ActivityInfo)field.get(this);
            o.screenOrientation = -1;
            field.setAccessible(false);
            return true;
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    private boolean isTranslucentOrFloating(){
        boolean isTranslucentOrFloating = false;
        try {
            int [] styleableRes = (int[]) Class.forName("com.android.internal.R$styleable").getField("Window").get(null);
            final TypedArray ta = obtainStyledAttributes(styleableRes);
            Method m = ActivityInfo.class.getMethod("isTranslucentOrFloating", TypedArray.class);
            m.setAccessible(true);
            isTranslucentOrFloating = (boolean)m.invoke(null, ta);
            m.setAccessible(false);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return isTranslucentOrFloating;
    }

    @Override
    public void setRequestedOrientation(int requestedOrientation) {
        if (Build.VERSION.SDK_INT == Build.VERSION_CODES.O && isTranslucentOrFloating()) {
            return;
        }
        super.setRequestedOrientation(requestedOrientation);
    }
'''
# android 9.0 （api 28）

----------

> 不能再使用http 请求
## 解决方法一
- 创建network_security_config.xml
<pre>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
</pre> 
- 在AndroidManifest.xml声明
<pre>
<application
...
 android:networkSecurityConfig="@xml/network_security_config"
...
/>
</pre>
## 解决方法二
- 直接声明使用明文流量 
<pre>
<application
...
  android:usesCleartextTraffic="true"
...
/>
 
</pre>