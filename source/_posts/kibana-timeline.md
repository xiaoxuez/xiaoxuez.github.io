---
title: kibana-timeline
date: 2017-10-14 13:54:37
categories:
- elk
---
在路径..kibana-5.4.2/src/core_plugins/timelion下为timeline的源码。

### fit-functitons

fit方法有average,carry,nearest,scale几种。

#### average

average的方法参数有2，dataTuples, targetTuples。前者为数据数据，后者为目标数据，数据结构都为[[time,value],[..]],目标数据的设定为以时间分的桶，传入时数据为[time,null]..

方法作用一，遍历时间桶，取数据中有在当前桶时间的范围内的数据，求平均值，若桶内无数据，则记为NaN,结果记为resultValues，随后再进行NaN处理，将resultValues中所有NaN的值都取代为值，取值的函数为	取前一次有值的数，与当前的数的差值 除以 连续NaN的个数+1，得出这期间的增长率，再依次给其中连续的nan的值赋值为前一次值+增长率。最后resultValues与目标数据中取出来的时间桶组成返回值。

求平均值部分的代码如

```
  while (i < dataTuplesQueue.length && dataTuplesQueue[i][0] <= time) {
      avgSet.push(dataTuplesQueue[i][1]);
      i++;
    }
    dataTuplesQueue.splice(0, i);

    const sum = _lodash2.default.reduce(avgSet, function (sum, num) {
      return sum + num;
    }, 0);
    return avgSet.length ? sum / avgSet.length : NaN;
```

#### carry

方法参数有2，dataTuples, targetTuples。要求dataTuples的长度大于targetTuples的，即原本数据的时间桶分布密集。作用是在targetTuples时间桶间，返回dataTuples时间桶中的值。targetTuples的长度是小于dataTuples的，从算法上来看dataTuples多出去了的就没了...

#### nearest

时间桶间隔，取离自己最近的一个桶的值为值。

#### 结论

通过上述代码可见，fit方法提供的主要是对已分好的数据桶进行再加工，传入参数为原有数据桶，和 目标桶，目标桶提供了新的时间桶。

#### 测试新方法

在fit中添加一个新方法，为某个时间桶无值，则保持上一次的值。

```
'use strict';

var _lodash = require('lodash');

var _lodash2 = _interopRequireDefault(_lodash);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { default: obj }; }

// bug: 没考虑没值的情况，就是dataTuples为[].
module.exports = function (dataTuples, targetTuples) {
  return _lodash2.default.map(targetTuples, function (bucket) {
    const time = bucket[0];
    let i = 0;
    while (i < dataTuples.length - 1 && dataTuples[i + 1][0] < time) {
        i++;
    }
    const closest = dataTuples[i];
    dataTuples.splice(0, i);
    return [bucket[0], closest[1]];
  });
};
```

目前是添加到源码的fit_functions文件夹下，重启服务即可生效。

#### Question

以fit为例，timeline提供的其余处理数据的方法，如，mutilply等都是在原有数据桶进行操作，原有数据桶是通过.es生成，如何能控制原有数据桶呢？

### Timeline

- 创建一条曲线。

```
.es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct')

```

- 绘两条曲线，offset代表时间间隔,offset=-1h为前一个小时

```
.es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct'), .es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct')

```

- .label()为曲线添加描述，如两条曲线可分别添加增加可视化。

```
.es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('last hour'), .es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('current hour')

```

- title()方法为时序图添加标题。使用方法为添加到最后

```
.es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('last hour'), .es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('current hour').title('CPU usage over time')
```

- .lines为曲线设置appearance，.lines(fill=1,width=0.5)为填充1，宽度为1。默认曲线为填充0宽1

```
.es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('last hour').lines(fill=1,width=0.5), .es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('current hour').title('CPU usage over time')

```

- .color()为曲线设置颜色，包括其label,如color(gray)，也可直接使用颜色值color(#1E90FF)

```
.es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('last hour').lines(fill=1,width=0.5).color(gray), .es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('current hour').title('CPU usage over time').color(#1E90FF)
```

- .legend()设置位置和图例的样式。

> For this example, place the legend in the north west position of the visualization with two columns by appending .legend(columns=2, position=nw)

```
.es(offset=-1h,index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('last hour').lines(fill=1,width=0.5).color(gray), .es(index=metricbeat-*, timefield='@timestamp', metric='avg:system.cpu.user.pct').label('current hour').title('CPU usage over time').color(#1E90FF).legend(columns=2, position=nw)

```

- 使用数学计算

- max 取最大值, 使用在metric中，如metric=max:system.network.in.bytes
- derivative 取导数， .es返回对象的方法
- multiply 乘法， 前面的应该为数字序列
- divide 除法，前面的应该为数字序列

```
 .es(index=metricbeat*, timefield=@timestamp, metric=max:system.network.in.bytes).derivative().divide(1048576).lines(fill=2, width=1).color(green).label("Inbound traffic").title("Network traffic (MB/s)"), .es(index=metricbeat*, timefield=@timestamp, metric=max:system.network.out.bytes).derivative().multiply(-1).divide(1048576).lines(fill=2, width=1).color(blue).label("Outbound traffic").legend(columns=2, position=nw)

```

 //这个示例，是绘制出入网流，关心的是变化率，所以取导数，bytes to megabytes单位换算所以除以1024*1024，再则，出相对于入，在一副图中显示为增强视图感，便一个的值>0,一个<0来表示，故出的线乘-1

- 使用条件 if ()， 参数为(eq/ne.. , value, then do, else do)//then do、else do没有的就写null

- eq ==
- ne !=
- lt <
- lte <=
- gt >
- gte >=

```
 .es(index=metricbeat-*, timefield='@timestamp', metric='max:system.memory.actual.used.bytes'), .es(index=metricbeat-*, timefield='@timestamp', metric='max:system.memory.actual.used.bytes').if(gt,12500000000,.es(index=metricbeat-*, timefield='@timestamp', metric='max:system.memory.actual.used.bytes'),null).label('warning').color('#FFCC11'), .es(index=metricbeat-*, timefield='@timestamp', metric='max:system.memory.actual.used.bytes').if(gt,15000000000,.es(index=metricbeat-*, timefield='@timestamp', ='max:system.memory.actual.used.bytes'),null).label('severe').color('red')
```

 //这个示例呢，是大于12500000000的绘制一种，大于15000000000绘制一种，看得出来if必须作为.es返回对象的方法，故有两种不同判断就得写两个相同的.es。

- 趋势，取数据个数的窗口的平均值连线。对于消除时间连续来说是极好的选择

```
.es().if(lt, 500, null).if(gte, 500, 1000)
.es().if(lt, 500, 0, 1000)
```

```
- mvavg()，如mvavg（10）
```



- .bars, .lines, .point改变展现形式的，.point是圆形
