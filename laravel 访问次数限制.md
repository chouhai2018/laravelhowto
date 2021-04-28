# laravel 访问次数限制

## 定义访问次数限制器

Laravel包括功能强大且可自定义的访问次数限制服务，您可以利用这些服务来限制给指定路由或一组路由的访问次数。首先，您应该定义满足应用程序的配置。在configureRateLimiting应用程序App\Providers\RouteServiceProvider内完成。

访问次数限制器是使用RateLimiter立面的for方法定义的。该for方法接受一个访问次数限制器名称和一个闭包，该闭包返回应应用于分配给该访问次数限制器使用的路由限制配置。限制配置是Illuminate\Cache\RateLimiting\Limit该类的实例。此类包含有用的“生成器”方法，以便您可以快速定义限制。

**用例：**操作文件App\Providers\RouteServiceProvider

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

/*
*访问次数限制器配置
*/
protected function configureRateLimiting()
{
	RateLimiter::for('global', function (Request $request) {
	//每分钟访问1000次
	return Limit::perMinute(1000);
	});
}
```

**注明:限制的使用方法**

**Limit::perMinute(1000)  //每分钟内最多访问1000次**

**Limit::perMinutes(20,1000)  //20分钟内最多访问1000次**

**Limit::perHour //每小时内最多访问1000次**

**Limit::perHour(1000,50) //50小时内最多访问1000次**

**Limit::perDay(1000)  //每天最多访问1000次**

如果传入请求超出指定的限制，Laravel将自动返回带有429 HTTP状态代码的响应。如果您想定义自己的限制返回的响应，则可以使用以下`response`方法：

```php
RateLimiter::for('global', function (Request $request) {
    return Limit::perMinute(1000)->response(function () {
        return response('Custom response...', 429);
    });
});
```

**注明:global为全部路由限制**

由于限制器回调收到传入的HTTP请求实例，因此您可以根据传入的请求或经过身份验证的用户动态构建适当的限制：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
                ? Limit::none()
                : Limit::perMinute(100);
});
```

**注明:upload为针对upload路由限制**

## 细分访问限制

例如，您可能希望允许用户每个IP地址每分钟100次访问给定的路由。为此，您可以在建立限制时使用以下方法：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()->vipCustomer()
                ? Limit::none()
                : Limit::perMinute(100)->by($request->ip()); //用户访问ip地址
});
```

为了用另一个示例说明此功能，我们可以将访客的访问权限限制为每个经过身份验证的用户ID每分钟100次或每IP地址每分钟10次：

```php
RateLimiter::for('uploads', function (Request $request) {
    return $request->user()
                ? Limit::perMinute(100)->by($request->user()->id) //用户ip
                : Limit::perMinute(10)->by($request->ip()); //用户访问ip地址
});
```

注明:$request内容太多了,不一一列出了,编辑器都列出了,根据需要选择就好.

## 多个访问限制

如果需要，您可以返回给定速率限制器配置的速率限制数组。将根据路由中每个速率限制在数组中的放置顺序对其进行评估：

```php
RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')), //email输入框
    ];
});
```



## 将访问限制器附加到指定路由

可以使用`throttle` [中间件](https://laravel.com/docs/8.x/middleware)将访问限制器附加到路由或路由组。中间件接受您希望分配给路由：

```php
Route::middleware(['throttle:uploads'])->group(function () {
    Route::post('/audio', function () {
        //
    });

    Route::post('/video', function () {
        //
    });
});
```

**附录:throttle安装和简单使用**

```shell
$ composer require "graham-campbell/throttle:^8.1"
```

在config/app.php中添加

```php
 'Throttle' => GrahamCampbell\Throttle\Facades\Throttle::class,
```

执行

```shell
$ php artisan vendor:publish
```

例子:

```php
/*
*  中间件匹配的路由30分钟内最多访问50次
*/

use Illuminate\Support\Facades\Route;

Route::get('foo', ['middleware' => 'GrahamCampbell\Throttle\Http\Middleware\ThrottleMiddleware:50,30', function () {
    return 'Why herro there!';
}]);
```

这里就简述下,改天写个Throttle全指南

## Redis支持

通常，`throttle`中间件被映射到`Illuminate\Routing\Middleware\ThrottleRequests`该类。此映射在应用程序的HTTP内核（`App\Http\Kernel`）中定义。但是，如果您将Redis用作应用程序的缓存驱动程序，则可能希望更改此映射以使用`Illuminate\Routing\Middleware\ThrottleRequestsWithRedis`该类。此类在使用Redis管理速率限制方面更为有效：

```php
'throttle' => \Illuminate\Routing\Middleware\ThrottleRequestsWithRedis::class,
```