## SDWebImage

1、SDwebImage

(1)图片会强制解压缩

(2)对大图会进行裁剪(是超过固定值进行裁剪)

(3)存储默认是存储到磁盘和内存中,其中存储到磁盘是使用的串行队列.(所以用在列表里,初次下载图片会卡顿)

2、AFNetworking

(1)存储图片是只存到内存中,存储到内存使用的是栅栏.(性能会好点,用在列表里不会存在卡顿)

(2)图片会进行解压缩

