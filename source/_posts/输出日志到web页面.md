---
title: 输出日志到web页面
date: 2016-03-27 17:26:08
tags: python
---

发布工具中需要对应用的启动日志，输出到页面上，方便用户查看。于是写了一个简单的python应用，基于diango。 先看看效果：
![image](/../../gallery/logy_1.png)

定义一个django view来接收ajax请求：
```python

def index(request):
	return render(request, 'index.html')

def pull(request):
	if request.method == 'GET':
		file_path = request.GET.get('file_path')
		if not file_path or len(file_path)==0:
			return HttpResponse('illegal file_path parameter')
		if not os.path.exists(file_path):
			return HttpResponse('file path: ' + file_path + ' does not exists, pls check!')
		if not os.path.isfile(file_path):
			return HttpResponse('file path: ' + file_path + ' is not file, pls check!')
		begin = request.GET.get('begin', 1)
		offset = request.GET.get('offset', 10)
		cmd = "sed -n '{0},{1}p' {2}".format(str(begin), str(begin + offset), file_path)
		result = commands.getstatusoutput(cmd)
		content = result[1]
		response_data = {}
		if content:
			response_data['data'] = content.splitlines()
			response_data['length'] = len(response_data['data'])
		else:
			response_data['length'] = 0
			
		return HttpResponse(json.dumps(response_data), content_type="application/json")
	else:
		return HttpResponse('illegal http method')
```

前端很简单通过开始行，offset向服务器请求日志数据，并且append到页面就可以了
```html
<html>
<head>
<title>Logly</title>
<script type="text/javascript" src="/static/js/jquery.min.js"></script>
<style type="text/css"> 
	html,body{
		background:#000; color:#090;
	} 
</style>
<script type="text/javascript">
$(document).ready(function(){
	var querystring = location.search.replace( '?', '' ).split( '&' );
	var queryObj = {};
	for ( var i=0; i<querystring.length; i++ ) {
      var name = querystring[i].split('=')[0];
      var value = querystring[i].split('=')[1];
      queryObj[name] = value;
	}
	var file_path = queryObj['file_path'];
	var begin = 1;
	var offset = 10;
	if(!file_path){
		$('#log').append('<p> file_path parameter is missed. <br />' +
		'eg: polls/index?file_path=urlencode(path) </p>'); 
	} else {
		var timer = setInterval(function () {
			var url = 'http://'+location.host + '/polls/pull?file_path=' + file_path + 
			'&begin=' + begin +
			'&offset=' + offset;
			$.ajax({
				url: url, 
				type: 'GET', 
				success: function(ret){
					if(ret==500){
						clearInterval(timer);
					}
					if(ret.length > 0){
						ret.data.forEach(function(line){
							$('#log').append(line + '<br />');
							begin ++;
						});
	                     window.scrollTo(0,document.body.scrollHeight); 
					 }
				}, error: function(ret){
					$('#log').append( + 'server error !! contact logly admin<br />');
					return false; 
				}
			});
		}, 500);
	}
});
</script>
</head>

<body>
<div style="margin-top:10px;"> 
	<p id='log'></p> 
</div> 
</body>
</html>
```
[Demo源代码](https://github.com/acupple/logly)