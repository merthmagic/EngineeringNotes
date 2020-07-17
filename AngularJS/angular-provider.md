## AngularJs Provider ##

Angular应用中大部分对象的initialize和wired工作是由`injector service`来完成的.

`injector`创建两种类型的对象:`services`和`specialized objects`.

`Services`对象提供用户定义的API，`specialized objects`提供angular的API,包括`controllers`,`directives`,`filters`,`animations`.

`injetctor`需要知道如何去构建对象，所以需要在应用中注册这些构建的信息，一共有五种recipe可以用于注册:`Value`,`Provider`,`Factor`,`Service`,`Constant`.只有`Provider`是真正用于注册的，其他四种都是基于Provider的语法糖.

### Value ###
Value这种方式是最简单的服务注册.

	var myApp = angular.module('myApp', []);
	myApp.value('clientId', 'a12345654321x');

### Factory ###
Value这种方式过于简单，并且有些功能没法实现，Factory这种方式与Value相比要有如下的优点:
1. 能够使用其他的Service
2. 能够进行service的初始化工作
3. 能够进行延迟初始化(delayed/lazy)

官方示例代码:

	myApp.factory('apiToken', ['clientId', function apiTokenFactory(clientId) {
	  var encrypt = function(data1, data2) {
	    // NSA-proof encryption algorithm:
	    return (data1 + ':' + data2).toUpperCase();
	  };
	
	  var secret = window.localStorage.getItem('myApp.secret');
	  var apiToken = encrypt(clientId, secret);
	
	  return apiToken;
	}]);

这里官方提到了一个Best practice，把定义serviced的function名字定义为<serviceId>Factory(方便追踪堆栈).

### Service ###
Service Recipe返回一个自定义js type的实例.
定义一个JS Type
	
	function UnicornLauncher(apiToken) {
	
	  this.launchedCount = 0;
	  this.launch = function() {
	    // Make a request to the remote API and include the apiToken
	    ...
	    this.launchedCount++;
	  }
	}

要返回一个该类型的实例可以用`Factory Recipe`

	myApp.factory('unicornLauncher', ["apiToken", function(apiToken) {
	  return new UnicornLauncher(apiToken);
	}]);

但service语法糖可以把这件事做的更漂亮

	myApp.service('unicornLauncher', ["apiToken", UnicornLauncher]);

这个recipe可以参照`constructor inject`这个模式来理解.

### Provider ###

这个recipe才是真正的定义service的方法，有最全的功能，但部分功能可能很少用到，所以才有了那几个语法糖.一般在需要在app启动前对service进行配置的情况下，才会用这种方法定义service.

这种方法实际上是定义了一个js类型，这个类型实现了`$get`方法，`$get`方法是一个工厂方法，和`Factory Recipe`的效果一样(实际上在用factory方法定义服务时，angular会自动定义一个空的provider).

	myApp.provider('unicornLauncher', function UnicornLauncherProvider() {
	  var useTinfoilShielding = false;
	
	  this.useTinfoilShielding = function(value) {
	    useTinfoilShielding = !!value;
	  };
	
	  this.$get = ["apiToken", function unicornLauncherFactory(apiToken) {
	
	    // let's assume that the UnicornLauncher constructor was also changed to
	    // accept and use the useTinfoilShielding argument
	    return new UnicornLauncher(apiToken, useTinfoilShielding);
	  }];
	});
	
	//配置服务
	myApp.config(["unicornLauncherProvider", function(unicornLauncherProvider) {
	  unicornLauncherProvider.useTinfoilShielding(true);
	}]);

### Constant ###
`angular`把生命周期分为配置阶段和运行阶段，而配置函数在配置阶段中被运行，这一阶段中没有服务可用(甚至是最简单的Value Recipe的亦不可用).

对于一些简单的值，如url前缀，是没有什么依赖或者配置的，因此经常需要在配置阶段和运行阶段都可使用，这一功能就由`Constant Recipe`来完成.

	myApp.constant('planetName', 'Greasy Giant');

使用constant

	myApp.config(['unicornLauncherProvider', 'planetName', function(unicornLauncherProvider, planetName) {
	  unicornLauncherProvider.useTinfoilShielding(true);
	  unicornLauncherProvider.stampText(planetName);
	}]);
	
	myApp.controller('DemoController', ["clientId", "planetName", function DemoController(clientId, planetName) {
	  this.clientId = clientId;
	  this.planetName = planetName;
	}]);


#### Reference ####
[1]https://docs.angularjs.org/guide/providers