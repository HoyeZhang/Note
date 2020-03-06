> 使用Databinding和ViewModel，以及LiveData构建一个小demo
# 配置
## 配置gradle.properties
<pre>
android.useAndroidX=true
android.enableJetifier=true
</pre>
## 配置Gradle 
<pre>
android {
    ...
    dataBinding{
        enabled true
    }
}
</pre>
## 添加依赖
<pre>
dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation 'androidx.appcompat:appcompat:1.1.0-alpha03'
    implementation 'androidx.constraintlayout:constraintlayout:2.0.0-alpha3'
    implementation 'com.google.android.material:material:1.1.0-alpha04'
    testImplementation 'junit:junit:4.12'
    androidTestImplementation 'androidx.test:runner:1.1.1'
    androidTestImplementation 'androidx.test.espresso:espresso-core:3.1.1'
    implementation 'com.github.bumptech.glide:glide:4.8.0'
    def lifecycle_version = "2.0.0"
    // ViewModel and LiveData
    implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"

}
</pre>
# 使用开发
## ViewModel类
<pre>
import androidx.lifecycle.MutableLiveData;
import androidx.lifecycle.ViewModel;

public class MainViewModel extends ViewModel {

    private MutableLiveData<String> test=new MutableLiveData<>();

    public MutableLiveData<String> getTest() {
        return test;
    }

    public void setTest(String test) {
        this.test.setValue(test);
    }
}
</pre>
- MainViewModel继承自ViewModel类，里面定义了一个LiveData数据，并添加了两个方法
## activity_main.xml

<?xml version="1.0" encoding="utf-8"?>

<layout
    xmlns:android="http://schemas.android.com/apk/res/android">

    <data>

        <variable
            name="viewModel"
            type="com.yk.jchat.MainViewModel"/>

    </data>

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:text="@{viewModel.test}"/>

    </LinearLayout>

</layout>
 # MainActivity
<pre>
package com.yk.jchat;

import android.os.Bundle;

import com.yk.jchat.databinding.ActivityMainBinding;

import androidx.appcompat.app.AppCompatActivity;
import androidx.databinding.DataBindingUtil;
import androidx.lifecycle.ViewModelProviders;

public class MainActivity extends AppCompatActivity {

    ActivityMainBinding mBinding;

    MainViewModel mViewModel;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mBinding = DataBindingUtil.setContentView(this,R.layout.activity_main);
        mViewModel= ViewModelProviders.of(this).get(MainViewModel.class);
        mBinding.setViewModel(mViewModel);
        mBinding.setLifecycleOwner(this);

        mViewModel.setTest("Hello World");
    }
}
</pre>
- onCreate里的方法和我们平时的方法有点不一样，首先加载布局不是用的setContentView，而是用的DataBindingUtil.setContentView()方法，并且这个方法还返回了ActivityMainBinding实例

- ActivityMainBinding这个类从哪来的呢，其实这个类是自动生成的，命名的方式就是布局activity_main去掉 “_” 后首字母大写后再加上Binding
- ActivityMainBinding这个类就可以为其布局里的data赋值，比如我们调用了mBinding.setViewModel(mViewModel)，只要我们在布局的data节点里写了variable，它就能为生成seter和geter方法，我们就对其进行相应的操作
- 相比于之前通过findViewBy等操作，现在的代码看起来简洁了许多，不光看着简洁，用起来也十分的方便，因为我们使用了一个方法，mBinding.setLifecycleOwner(this)，使用之后，只要我们的数据发生了改变，UI相应的也会变化，比如我们在之后加上一句mViewModel.setTest("Hello World")，这样我们的TextView就会显示出Hello World字样
