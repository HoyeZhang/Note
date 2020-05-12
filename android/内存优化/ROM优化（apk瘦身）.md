# apk分析
- 打开android studio下的analyze apk，选择当前的release apk （或直接将apk拖进Android Studio即可 ）
- 打开后的apk结构
	- dex 
	- res
	- asserts
	- lib
	- AndroidManifest
	- resources.arsc : 编译后的二进制资源文件
- 瘦身完毕可通过分析工具的compare with进行前后对比
# apk瘦身
## 资源
- shrinkResources  去除无用资源
### 重复资源
- 在大型 App 项目的开发中，一个 App 一般会有多个业务团队进行开发，其中每个业务团队在资源提交时的资源名称可能会有重复的，这将会 引发资源覆盖的问题，因此，每个业务团队都会为自己的 资源文件名添加前缀。

- 这样就导致了这些资源文件虽然 内容相同，但因为 名称的不同而不能被覆盖，最终都会被集成到 APK 包中。这里，我们还是可以 在 Android 构建工具执行 package${flavorName}Task 之前通过修改 Compiled Resources 来实现重复资源的去除，具体放入实现原理可细分为如下三个步骤：

	1）首先，通过资源包中的每个ZipEntry的CRC-32 checksum来筛选出重复的资源。  
	2）然后，通过android-chunk-utils修改resources.arsc，把这些重复的资源都重定向到同一个文件上。  
	3）最后，把其它重复的资源文件从资源包中删除，仅保留第一份资源。
<pre>

variantData.outputs.each {
    def apFile = it.packageAndroidArtifactTask.getResourceFile();

    it.packageAndroidArtifactTask.doFirst {
        def arscFile = new File(apFile.parentFile, "resources.arsc");
        JarUtil.extractZipEntry(apFile, "resources.arsc", arscFile);

        def HashMap<String, ArrayList<DuplicatedEntry>> duplicatedResources = findDuplicatedResources(apFile);

        removeZipEntry(apFile, "resources.arsc");

        if (arscFile.exists()) {
            FileInputStream arscStream = null;
            ResourceFile resourceFile = null;
            try {
                arscStream = new FileInputStream(arscFile);

                resourceFile = ResourceFile.fromInputStream(arscStream);
                List<Chunk> chunks = resourceFile.getChunks();

                HashMap<String, String> toBeReplacedResourceMap = new HashMap<String, String>(1024);

                // 处理arsc并删除重复资源
                Iterator<Map.Entry<String, ArrayList<DuplicatedEntry>>> iterator = duplicatedResources.entrySet().iterator();
                while (iterator.hasNext()) {
                    Map.Entry<String, ArrayList<DuplicatedEntry>> duplicatedEntry = iterator.next();

                    // 保留第一个资源，其他资源删除掉
                    for (def index = 1; index < duplicatedEntry.value.size(); ++index) {
                        removeZipEntry(apFile, duplicatedEntry.value.get(index).name);

                        toBeReplacedResourceMap.put(duplicatedEntry.value.get(index).name, duplicatedEntry.value.get(0).name);
                    }
                }

                for (def index = 0; index < chunks.size(); ++index) {
                    Chunk chunk = chunks.get(index);
                    if (chunk instanceof ResourceTableChunk) {
                        ResourceTableChunk resourceTableChunk = (ResourceTableChunk) chunk;
                        StringPoolChunk stringPoolChunk = resourceTableChunk.getStringPool();
                        for (def i = 0; i < stringPoolChunk.stringCount; ++i) {
                            def key = stringPoolChunk.getString(i);
                            if (toBeReplacedResourceMap.containsKey(key)) {
                                stringPoolChunk.setString(i, toBeReplacedResourceMap.get(key));
                            }
                        }
                    }
                }

            } catch (IOException ignore) {
            } catch (FileNotFoundException ignore) {
            } finally {
                if (arscStream != null) {
                    IOUtils.closeQuietly(arscStream);
                }

                arscFile.delete();
                arscFile << resourceFile.toByteArray();

                addZipEntry(apFile, arscFile);
            }
        }
    }
}

</pre>
### libs
- libs下有armeabi，armeabi-v7a，x86等架构
	- armeabi 第5代 ARM v5TE，使用软件浮点运算，兼容所有ARM设备，通用性强，速度慢（只支持armeabi）
	- armeabi-v7a 第7代 ARM v7，使用硬件浮点运算，具有高级扩展功能（支持 armeabi 和 armeabi-v7a，目前大部分手机都是这个架构）
	- arm64-v8a 第8代，64位，包含AArch32、AArch64两个执行状态对应32、64bit（支持 armeabi-v7a、armeabi 和 arm64-v8a）
	- *x86 intel 32位，一般用于平板（支持 armeabi(性能有所损耗) 和 x86）
	- x86_64 intel 64位，一般用于平板（支持 x86 和 x86_64）
	- mips 基本没见过（支持 mips）
	- mips64 基本没见过（支持 mips 和 mips_64）
- 当一个应用安装在设备上，只有该设备支持的CPU架构对应的.so文件会被安装。不同CPU架构的Android手机加载时会在libs下找自己对应的目录，从对应的目录下寻找需要的.so文件；如果没有对应的目录，就会去armeabi下去寻找，如果已经有对应的目录，但是如果没有找到对应的.so文件，也不会去armeabi下去寻找了。
- 以x86设备为例，x86设备会在项目中的 libs文件夹寻找是否含有x86文件夹，如果含有x86文件夹，则默认为该项目有x86对应的so可运行文件，只有x86文件夹而文件夹下没有so，程序运行也是会出现findlibrary returned null的错误的；如果工程本身不含有x86文件夹，则会寻找armeabi或者armeabi-v7a文件夹，兼容运行。以armeabi-v7a设备为例，该Android设备当然优先寻找libs目录下的armeabi-v7a文件夹，同样，如果只有armeabi-v7a文件夹而没有 so也是会报错的；如果找不到armeabi-v7a文件夹，则寻找armeabi文件夹，兼容运行该文件夹下的so，但是不能兼容运行x86的so。所以项目中如果只含有x86的so，在armeabi和armeabi-v7a也是无法运行的。以上就是不同CPU架构运行时加载so的策略。
- 第三方的类库只提供了armeabi下的.so文件，我们项目里适配了armeabi-v7a和x86，如果不在对应的文件下放对应的.so文件，就可能导致某些Android设备会出一些问题，我们可以复制armeabi下得.so文件到不同的文件夹下。如果第三方提供了不同平台的.so文件，则复制不同平台的.so文件到项目中对应的文件夹下即可。
#### so瘦身
- armeabi 目录下的 So 可以兼容别的平台上的 So，相当于是一个万金油，都可以使用。但是，这样 别的平台使用时性能上就会有所损耗，失去了对特定平台的优化。
- 想要完美支持所有类型的设备代价太大，那么，我们能不能采取一个 折中的方案，就是 对于性能敏感的模块，它使用到的 So，我们都放在 armeabi 目录当中随着 Apk 发出去，然后我们在代码中来判断一下当前设备所属的 CPU 类型，根据不同设备 CPU 类型来加载对应架构的 So 文件。
<pre>
String abi = "";
// 获取当前手机的CPU架构类型
if (Build.VERSION.SDK_INT < Build.VERSION_CODES.LOLLIPOP) {
    abi = Buildl.CPU_ABI;
} else {
    abi = Build.SUPPORTED_ABIS[0];
}

if (TextUtils.equals(abi, "x86")) {
    // 加载特定平台的So

} else {
    // 正常加载

}
</pre>
- so 文件动态下发 [https://mp.weixin.qq.com/s/X58fK02imnNkvUMFt23OAg](https://mp.weixin.qq.com/s/X58fK02imnNkvUMFt23OAg "动态下发 so 库在 Android APK 安装包瘦身方面的应用")

###  图片
- 压缩（发版前通过Tiny_Gradle_plugin插件使用tintpng压缩图片，app默认打包会进行图片压缩的工作，此时建议手动禁止，android{defaultConfig{...}aaptOptions{cruncherEnadles = false}）
- webp格式（兼容性不好，最低android4.2.1，不便于预览，须使用浏览器打开）
- 大图保留xhdpi ，logo等可以保留hdpi，xhdpi

### assets
- 删除无用资源
## 代码
- 复用及统一按压效果
- item统一selecctor
- 开启混淆
- 插件化


# 参考
- https://mp.weixin.qq.com/s/uENpxB7Ns_x-5E2pTHwFdQ