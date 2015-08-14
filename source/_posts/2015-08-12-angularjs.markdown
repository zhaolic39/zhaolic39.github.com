---
layout: post
title: "angularjs学习记录"
date: 2015-08-12 18:18:45 +0800
comments: true
categories: web前端
---
## Angularjs
虽然在公司项目上用到了angularjs，但还有很多不了解的地方。利用台风的放假时间学习了一下`AngularJS权威教程`，这里简单记录一下概念与要点。

### 基础
#### Module 模块
`module`是按业务划分的模块，属于逻辑上的概念，一个`module`下可以包含多个`sevice`,多个`controller`,自己的`router`等。

#### Controller 控制器
`controller`是与模版页面交流的代码，html可以直接访问`#scope`上的对象、属性、函数。
一般在`controller`中不含有复杂的业务逻辑，例如接口调用的代码应该放在`service`层中。
`controller`的生命周期短，在刷新或跳转时就会消失，所以需要保持的数据应该放在`service`中。

#### Service 服务
一般把接口调用，复杂逻辑代码，持久数据放在`service`中。
定义`service`使用`factory`函数，例：
```javascript
//在myApp模块中定义UserService
angular.module('myApp', [])
    .factory('UserService', function(){
        var userInfo;
        return {
            getUserInfo : funciton(){
                return userInfo;
            }
        };
    })
```
`controller`可以通过依赖注入的方式使用`service`，例：
```javascript
//UserService是通过注入进来
.controller('ServiceController', funciton($scope, UserService){
    $scope.doSomething = function(){
        return UserService.getUserInfo();
    };
})
```
#### filter 过滤器
使用过滤器可以实现页面输出的字典转换，例：
```javascript
{{ name | uppercase }} //使用内置过滤器uppercase将name转为大写
```
也可以自定义过滤器，实现业务上的输出转换，例：
```javascript
angular.module('myApp.filters', [])
    .filter('capitalize', function() {
        return function(input) {    //input是要转换的原始输入
            return '';               //返回过滤后的输出
        };
    });
```

### 扩展
#### Restangular 替代\$http和$resource
比原生好用的restful接口访问库，[mgonto/restangular][1]，一些例子：
```javascript
// GET to http://www.google.com/1 You set the URL in this case
Restangular.oneUrl('googlers', 'http://www.google.com/1').get();
// GET /accounts/123/messages
Restangular.one("accounts", 123).customGET("messages")
// GET /accounts/messages?param=param2
Restangular.all("accounts").customGET("messages", {param: "param2"})
// POST /accounts/123/messages?param=myParam with the body of name: "My Message"
account.customPOST({name: "My Message"}, "messages", {param: "myParam"}, {})
```
#### ui-router 替代ng-router
使用状态机组织的路由框架，可以实现在路由的页面中再次跳转的嵌套路由。例：
```javascript
$stateProvider
    .state('inbox', {
        url: '/inbox/:inboxId',
        template: '<div><h1>Welcome to your inbox</h1>\
            <a ui-sref="inbox.priority">Show priority</a>\
            <div ui-view></div>\
            </div>'
        controller: function($scope, $stateParams) {
                $scope.inboxId = $stateParams.inboxId;
            }
        })
    .state('inbox.priority', {
        url: '/priority',
        template: '<h2>Your priority inbox</h2>'
    });
```
* /inbox/1匹配第一个状态。
* /inbox/1/priority匹配第二个状态。
  [1]: https://github.com/mgonto/restangular
