---
category: algorithm
tags:
  - php
  - algorithm
date: 2020-09-08
title: 轨迹抽稀之道格拉斯-普克算法
vssue-title: PHP implementation of the Ramer Douglas Peucker Algorithm
---
## 缘起

最近项目涉及到车辆轨迹展示，由于车辆上传轨迹点到频率为每5秒为一个点，有时活跃车辆一天到轨迹点有1万之多，不仅接口返回数据慢，同时前端页面渲染的速度也受到了影响。所以需要一个算法来对轨迹点进行压缩。

我们知道，运动轨迹是由多个经纬度坐标点组合而成。那么这么多的经纬度坐标点是否每个都需要呢？肯定不是，因为有很多的坐标点是重合的，比如车辆禁止状态上传的点，或者在一条直线上的点其实只需要两个点就可以表示一条路径了。实际上将这些多余的点剔除仍然能保证轨迹曲线大体不变，并且还能节省存储空间，这样的过程我们称之为抽稀。

道格拉斯-普克算法就是其中的一种实现算法。

> 道格拉斯-普克算法(Douglas–Peucker algorithm，亦称为拉默-道格拉斯-普克算法、迭代适应点算法、分裂与合并算法)是将曲线近似表示为一系列点，并减少点的数量的一种算法。该算法的原始类型分别由乌尔斯·拉默（Urs Ramer）于1972年以及大卫·道格拉斯（David Douglas）和托马斯·普克（Thomas Peucker）于1973年提出，并在之后的数十年中由其他学者予以完善。

## 实现

### 算法逻辑

算法的具体实现逻辑如下：

1. 在轨迹曲线在曲线首尾两点 A，B 之间连接一条直线 AB，该直线为曲线的弦；
2. 遍历曲线上其他所有点，求每个点到直线AB的距离，找到最大距离的点 C，最大距离记为 maxDistance
3. 比较该距离 maxDistance 与预先定义的阈值 epsilon 大小，如果 maxDistance < epsilon，则将该直线 AB 作为曲线段的近似，舍去 AB 之间的所有点，曲线段处理完毕；
4. 若 maxDistance >= epsilon，则使 C 点将曲线 AB 分为 AC和 CB 两段，并分别对这两段进行（1）~（3）步处理；
5. 当所有曲线都处理完毕时，依次连接各个分割点形成的折线，即为原始曲线的路径。

![Douglas–Peucker](https://public.zulu.wang/img/20200908180657.gif)

### 相关代码

```php
class DouglasPeucker
{
    public static function perpendicularDistance($p, $p1, $p2)
    {
        if ($p1[0] == $p2[0]) {
            return abs($p[0] - $p1[0]);
        }
        $slope = ($p2[1] - $p1[1]) / ($p2[0] - $p1[0]);
        $intercept = $p1[1] - ($slope * $p1[0]);
        return abs($slope * $p[0] - $p[1] + $intercept) / sqrt(pow($slope, 2) + 1);
    }


    public static function RDP($points, $epsilon)
    {
        $firstPoint = reset($points);
        $lastPoint = end($points);
        if (count($points) < 3) {
            return $points;
        }
        $index = 0;
        $maxDistance = 0;
        foreach ($points as $currentIndex => $point) {
            $distance = self::perpendicularDistance($point, $firstPoint, $lastPoint);
            if ($distance > $maxDistance) {
                $maxDistance = $distance;
                $index = $currentIndex;
            }
        }
        if ($maxDistance > $epsilon) {
            $left = self::RDP(array_slice($points, 0, $index + 1), $epsilon);
            $right = self::RDP(array_slice($points, $index, count($points) - 1), $epsilon);
            return array_merge(array_slice($left, 0, count($left) - 1), $right);
        } else {
            return [$firstPoint, $lastPoint];
        }
    }
}
```
## 测试

一共 812 个轨迹点，分多次设置 epsilon 的值，每次递增，结果如下
|  epsilon   | 轨迹点  |
|  ----  | ----  |
| 0.000001  | 676 |
| 0.00001  | 569 |
|  0.0001 | 250 |
| 0.001  | 35 |

![原始点](https://public.zulu.wang/img/20200909091702.png)

![epsilon = 0.000001](https://public.zulu.wang/img/20200909092004.png)

![epsilon = 0.00001](https://public.zulu.wang/img/20200909092129.png)

![epsilon = 0.0001](https://public.zulu.wang/img/20200909092330.png)

![epsilon = 0.001](https://public.zulu.wang/img/20200909092439.png)

截图从上至下分别是原始点以及 epsilon 分别是 0.000001、0.00001、0.0001、0.001 所形成点运动轨迹情况，可以看到当 epsilon 为 0.001 时，在一些拐角会有些许差异，但是整体运动轨迹相对平滑，于原始路径并无较大差异，仅用4%的点就可以展示大致路径，这个压缩率还是很高的，在传输及存储都显著的降低了成本。

### 路径显示代码

以下是在百度地图上绘制轨迹的相关代码，ak 为百度地图开发平台申请的密钥，请自行替换

```javascirpt
<!DOCTYPE html>
<html>
<head>
	<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
	<meta name="viewport" content="initial-scale=1.0, user-scalable=no" />
	<style type="text/css">
		body, html,#allmap {width: 100%;height: 100%;overflow: hidden;margin:0;font-family:"微软雅黑";}
	</style>
	<script type="text/javascript" src="https://api.map.baidu.com/api?v=2.0&ak=XXXX"></script>
	<title>运动轨迹展示</title>
</head>
<body>
	<div id="allmap"></div>
</body>
</html>
<script type="text/javascript">
	var map = new BMap.Map("allmap");


	var jsonStr35 = "[[117.24931335449219,30.843416213989258],[117.24864959716797,30.860300064086914],[117.24909973144531,30.854249954223633],[117.24791717529297,30.860965728759766],[117.20303344726562,30.891067504882812],[117.19245147705078,30.900365829467773],[117.19293212890625,30.911149978637695],[117.18883514404297,30.92121696472168],[117.19176483154297,30.93156623840332],[117.20001983642578,30.94285011291504],[117.1968994140625,30.946500778198242],[117.1993179321289,30.94398307800293],[117.19636535644531,30.946733474731445],[117.18986511230469,30.946916580200195],[117.18401336669922,30.95021629333496],[117.18751525878906,30.947900772094727],[117.18395233154297,30.950183868408203],[117.181396484375,30.948165893554688],[117.1807632446289,30.9424991607666],[117.18046569824219,30.94646644592285],[117.18416595458984,30.95025062561035],[117.18948364257812,30.947032928466797],[117.1977310180664,30.945966720581055],[117.19989776611328,30.942115783691406],[117.19108581542969,30.930166244506836],[117.18871307373047,30.920984268188477],[117.19023132324219,30.917882919311523],[117.1805648803711,30.911550521850586],[117.18329620361328,30.91320037841797],[117.1805648803711,30.911550521850586],[117.18264770507812,30.912782669067383],[117.17884826660156,30.913084030151367],[117.18031311035156,30.911617279052734],[117.15933227539062,30.931283950805664],[117.15951538085938,30.932817459106445]]";


	var points = JSON.parse(jsonStr35);
	var baiduPoints = [];
	points.forEach(function (item, index, array) {
		baiduPoints.push(new BMap.Point(item[0], item[1]));
	});


	//初始化地图,设置城市和地图级别。
	map.centerAndZoom(new BMap.Point(points[0][0], points[0][1]), 14);

	var polyline = new BMap.Polyline(baiduPoints, {strokeColor:"blue", strokeWeight:6, strokeOpacity:0.5});  //定义折线
	map.addOverlay(polyline);     //添加折线到地图上
</script>
```

## 参考
* [Ramer–Douglas–Peucker algorithm](https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm)
* [Javascript implementation of the Ramer Douglas Peucker Algorithm](https://karthaus.nl/rdp/)