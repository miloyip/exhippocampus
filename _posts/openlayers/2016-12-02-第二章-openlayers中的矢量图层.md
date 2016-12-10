---
layout: page
title: Geoserver+Openlayers3系列 ：openlayers中的矢量图层
categories:
    - frontend
header:
    image_fullwidth: chess.jpg
    caption: Photograph by Drolexandre
    caption_url: http://commons.wikimedia.org/wiki/File:Echecs_chinois.JPG
---
---
popup，即气泡，在前端设计中特别容易见到，通常有两种情况

* 鼠标移动到某点，利用气泡展示信息；
* 鼠标点击某点，利用气泡展示信息 

#popup实例#
  在ol3中，为了添加一个气泡，需要使用Overlay，顾名思义，即是覆盖层的意思，也就是说创建的气泡都是作为map的覆盖层显示在地图上的。
###overlay解释###
  首先，查看一个overlay类的定义：

    /** 
	* @enum {string} 
	*/  
	ol.OverlayProperty = {  
	  ELEMENT: 'element',  
	  MAP: 'map',  
	  OFFSET: 'offset',  
	  POSITION: 'position',  
	  POSITIONING: 'positioning'  
	};  
  解释：

* element表示你需要展现的气泡元素；
* map 可以通过map.setOverlay来设置；
* position 则表示位置坐标；
###定义popup###
先看代码：

    <div id="popup" class="ol-popup">  
    	<a href="#" id="popup-closer" class="ol-popup-closer"></a>  
   		 <div id="popup-content" style="width:300px; height:120px;"></div>  
	</div>
解释：

*  首先，定义一个div，作为popup的容器；
* 在div中，包括两部分，一个链接（也就是关闭按钮）；
* 还有一个div则是作为显示的信息内容；

现在我们意见建立了div，接下来就是需要将div转换成一个overlay，再将该overlay在点击地图以后添加给map；
###创建overlay对象###
    var overlay = new ol.Overlay(/**@type(olx.OverlayOption)*/({
		element : document.getElementById("popup"),
		autoPan : true
	}))
上述基本将需要弹出的内容框定义好了，接下来我们需要给给Map绑定事件，从而触发popup，下面是代码：

    /** 
	 * Add a click handler to the map to render the popup. 
	 */  
	map.addEventListener('click', function(evt) {  
	  var coordinate = evt.coordinate;  
	  var hdms = ol.coordinate.toStringHDMS(ol.proj.transform(  
	      coordinate, 'EPSG:3857', 'EPSG:4326'));  
	  content.innerHTML = '<p>你点击的坐标是：</p><code>' + hdms + '</code>';  
	  overlay.setPosition(coordinate);  
	  map.addOverlay(overlay);  
	});  
从代码上来看，首先获取点击点的坐标，并形成所要显示的overlay层，最后将该层添加到地图上。

至此，popup简单的实例就完成了，但是官方给的例子，popup所用的CSS，还是值得我记录一下的，平时很少用纯的CSS来写“

    .ol-popup {  
        position: absolute;  //定位 
        background-color: white;  //背景白色
        -webkit-filter: drop-shadow(0 1px 4px rgba(0,0,0,0.2));  //滤镜功能，值得再写一篇blog
        filter: drop-shadow(0 1px 4px rgba(0,0,0,0.2));  
        padding: 15px;  
        border-radius: 10px;  
        border: 1px solid #cccccc;  
        bottom: 12px;  
        left: -50px;  
      }  
      .ol-popup:after, .ol-popup:before {  
        top: 100%;  
        border: solid transparent;  
        content: " ";  
        height: 0;  
        width: 0;  
        position: absolute;  
        pointer-events: none;  
      }  
      .ol-popup:after {  
        border-top-color: white;  
        border-width: 10px;  
        left: 48px;  
        margin-left: -10px;  
      }  
      .ol-popup:before {  
        border-top-color: #cccccc;  
        border-width: 11px;  
        left: 48px;  
        margin-left: -11px;  
      }  
      .ol-popup-closer {  
        text-decoration: none;  
        position: absolute;  
        top: 2px;  
        right: 8px;  
      }  
      .ol-popup-closer:after {  
        content: "✖";  
      }  
 
	<!DOCTYPE html>
	<html>
	<head>
		<title></title>
		<script type="text/javascript" src="js/ol.js"></script>
	</head>
	<body>
	<div id="map"></div>
	<script type="text/javascript">
		var map = new ol.Map({
			target : 'map',
			layers : [
				new ol.layer.Tile({
					source :new ol.source.OSM()
				})
			],
			view : new ol.View({
				center :[0,0],
				zoom : 1
			})
		});
	</script>
	</body>
	</html>
![shut up](shawn_ol_popup_001.jpg)