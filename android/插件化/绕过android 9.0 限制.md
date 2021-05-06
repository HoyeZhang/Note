

>  https://zhuanlan.zhihu.com/p/37819685

# 系统是如何实现这个限制的？
- 通过反射或者JNI访问非公开接口时会触发警告/异常等，那么不妨跟踪一下反射的流程，看看系统到底在哪一步做的限制（以下的源码分析大可以走马观花的看一下，需要的时候自己再仔细看）。我们从 `java.lang.Class.getDeclaredMethod(String)` 看起，这个方法在Java层 最终调用到了 `getDeclaredMethodInternal` 这个native方法，看一下这个方法的源码：
<pre>
 static jobject Class_getDeclaredMethodInternal(JNIEnv* env, jobject javaThis,
                                               jstring name, jobjectArray args) {
  ScopedFastNativeObjectAccess soa(env);
  StackHandleScope<1> hs(soa.Self());
  DCHECK_EQ(Runtime::Current()->GetClassLinker()->GetImagePointerSize(), kRuntimePointerSize);
  DCHECK(!Runtime::Current()->IsActiveTransaction());
  Handle<mirror::Method> result = hs.NewHandle(
      mirror::Class::GetDeclaredMethodInternal<kRuntimePointerSize, false>(
          soa.Self(),
          DecodeClass(soa, javaThis),
          soa.Decode<mirror::String>(name),
          soa.Decode<mirror::ObjectArray<mirror::Class>>(args)));
  if (result == nullptr || ShouldBlockAccessToMember(result->GetArtMethod(), soa.Self())) {
    return nullptr;
  }
  return soa.AddLocalReference<jobject>(result.Get());
}

</pre>
注意到那个 **ShouldBlockAccessToMember** 调用了吗？如果它返回 true，那么直接返回`nullptr`，上层就会抛 `NoSuchMethodXXX` 异常；也就触发系统的限制了。

# 应对策略
## 1 修改 runtime 的hidden_api_policy_
<pre>
hiddenapi::EnforcementPolicy GetHiddenApiEnforcementPolicy() const {
    return hidden_api_policy_;
}
</pre>
- 也就是说，返回的是 runtime 这个对象的一个成员。**如果我们直接修改内存，把这个成员设置为 kNoChecks**，那么不就达到目标了吗？
## 2 调用类的ClassLoader修改为 BootClassLoader
- classloader实际上是Class类的第一个成员，而这个`java.lang.Class`我们肯定是能拿到的，因此我们可以通过上面提到的**修改偏移的方式直接修改ClassLoader**，进而绕过限制。
- JVM规范中，一个int占4字节；在ART实现中，一个Java对象的引用占用4字节（不论是32位还是64位），因此 **classloader的偏移量为8**；我们拿到调用者的Class对象，在JNI层拿到对象的内存表示，直接把偏移量为8处置空（BootClassLoader在为null）即可。当然，如果你不想用JNI，Unsafe 也能满足这个需求。