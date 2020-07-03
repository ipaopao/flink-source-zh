### Flink数据通信

从数据源开始分析数据通信的整个过程，SourceFunction接口中的SourceContext内部接口SourceContext的collect()方法用于发射
数据，其实现类NonTimestampContext的collect()方法直接调用了output对象的collect方法，它是Output<StreamRecord<T>>类
型，它的实际类型是CountingOutput类型，这是一个包装类型，是对Output的包装，并在此基础上增加了收集元素数量的numRecordsOut
的Counter类型的监控变量，collect()方法中调用了numRecordsOut.inc()方法来对元素数量进行自增，从而实现了对收集元素数量的监
控。NoTimestampContext的CountingOutput封装的output的真正类型是RecordWriterOutput类型，其collect()方法会直接过滤
掉输出到其它旁路input的数据，而对于输出到非旁路input的数据则直接使用pushToRecordWriter()方法进行序列化代理，并将数据传递
给recordWriter。