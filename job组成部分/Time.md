Event time:
Ingesttion time:
processing time:
首先设置StreamTimeCharacteristic，这决定了source的行为，窗口使用哪种时间等等。
env.setStreamTimeCharacteristic(TimeCharacteristic.ProcessingTime);
