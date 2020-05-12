
# Bitmap的内存计算方式？


在已知图片的长和宽的像素的情况下，影响内存大小的因素会有资源文件位置和像素点大小。

像素点大小：


常见的像素点有：



ARGB_8888：4个字节

ARGB_4444、ARGB_565：2个字节



资源文件位置：


不同dpi对应存放的文件夹  
![](../images/dpiimage.webp)


比如一个一张图片的像素为180*180px，dpi(设备独立像素密度)为320，如果它仅仅存放在drawable-hdpi，则有：



横向像素点 = 180 * 320/240 + 0.5f = 240 px
纵向像素点 = 180 * 320/240 + 0.5f = 240 px


如果它仅仅存放在drawable-xxhdpi，则有：



横向像素点 = 180 * 320/480 + 0.5f = 120 px
纵向像素点 = 180 * 320/480 + 0.5f = 120 px


所以，对于一张180*180px的图片，设备dpi为320，资源图片仅仅存在drawable-hdpi，像素点大小为ARGB_4444，最后生成的文件内存大小为：



横向像素点 = 180 * 320/240 + 0.5f = 240 px
纵向像素点 = 180 * 320/240 + 0.5f = 240 px
内存大小 = 240 * 240 * 2 = 115200byte 约等于 112.5kb


建议阅读：

《Android Bitmap的内存大小是如何计算的？》

https://ivonhoe.github.io/2017/03/22/Bitmap&Memory/


# Bitmap的高效加载？


Bitmap的高效加载在Glide中也用到了，思路：



获取需要的长和宽，一般获取控件的长和宽。

设置BitmapFactory.Options中的inJustDecodeBounds为true，可以帮助我们在不加载进内存的方式获得Bitmap的长和宽。

对需要的长和宽和Bitmap的长和宽进行对比，从而获得压缩比例，放入BitmapFactory.Options中的inSampleSize属性。

设置BitmapFactory.Options中的inJustDecodeBounds为false，将图片加载进内存，进而设置到控件中。