# resourcemanager
- resourcemanager负责registerJobManager和requestSlot
- 以YarnResourceManager为例，yarn在实际工程中用的最多。
- YarnJobClusterEntrypoint作为Yarn启动一个flink job的入口。
- YarnApplicationMasterRunner负责启动YarnFlinkResourceManager和jobManager