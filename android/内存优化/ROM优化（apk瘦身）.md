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