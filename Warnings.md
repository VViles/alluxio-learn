#注意事项
----------------
最近把Alluxio用在生产环境下，遇到很多坑，现总结如下。spark版本是1.5.2，hadoop版本是1.5.2。

1. 版本最好是用1.1.0，这样就跳过了很多奇怪的问题了。
2. Alluxio配置底层文件系统时，需要添加：
>sc.hadoopConfiguration.set("fs.alluxio.impl", "alluxio.hadoop.FileSystem")
>sc.hadoopConfiguration.set("alluxio.user.file.writetype.default", "CACHE_THROUGH")

3. 在yarn上部署需要重现编译Alluxio的包。指定hadoop版本，指定用spark编译
