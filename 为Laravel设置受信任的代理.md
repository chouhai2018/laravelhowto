# 为Laravel设置受信任的代理

设置受信任的代理后，Laravel对反向代理（例如负载均衡器或缓存）正确生成URL，重定向，会话处理和登录。如果您的Web服务器位于负载平衡器（Nginx，HAProxy，Envoy，ELB / ALB等），HTTP缓存（CloudFlare，Squid，Varnish等）或其他中间（反向）代理之后，这将很有用。

## 这是如何运作的？

反向代理应用程序通常读一些HTTP报头，如X-Forwarded，X-Forwarded-For，X-Forwarded-Proto（多），客户端做出的HTTP请求。
注意：如果未设置这些标头，则应用程序代码将认为每个传入的HTTP请求都将来自代理。
Laravel（从技术上讲是Symfony HTTP基类）具有“可信代理”的概念，其中X-Forwarded只有在知道请求的源IP地址的情况下才使用这些头。换句话说，如果代理受信任，它仅信任那些标头。
该软件包为该选项创建了更简单的界面。您可以设置代理的IP地址（应用程序会看到该代理的IP地址，因此它可能是专用网络IP地址），X-Forwarded如果包含这些标头的HTTP记录来自受信任的代理，Symfony HTTP类将知道使用标头。

## 为什么如此重要？

一种非常常见的负载平衡方法是将https://请求发送到负载平衡器，但将http://请求发送到负载平衡器后面的应用程序服务器。

例如，您可以在浏览器中向发送请求https://example.org。负载平衡器进而可以将请求发送到位于的应用程序服务器http://192.168.1.23。

如果该服务器返回重定向或生成URL，该怎么办？用户的浏览器将获取包含http://192.168.1.23其中的重定向或HTML ，这显然是错误的。

发生的情况是该应用程序认为其主机名是192.168.1.23，而架构则是http://。它不知道最终客户端用于https://example.org其Web请求。

因此，应用程序需要知道读取X-Forwarded标头以获取正确的请求详细信息（schema https://，host example.org）。

Laravel / Symfony自动读取这些标头，但前提是将可信代理配置设置为“信任”负载均衡器/反向代理。

*注意：我们中的许多人都使用托管的负载均衡器/代理，例如AWS ELB / ALB等。我们不知道这些反向代理的IP地址，因此在这种情况下，您需要信任所有代理。*

*注意：折衷方案存在安全风险，即允许人们潜在地欺骗X-Forwarded标头。*

## 安装

laravel > 5.5 安装好后已经集成了
Laravel 5.0-5.4
composer require fideloper/proxy:^3.3
Laravel 4
composer require fideloper/proxy:^2.0

## 配置

1、发布

```shell
php artisan vendor:publish --provider="Fideloper\Proxy\TrustedProxyServiceProvider"
```

2、在app/Http/Kernel.php中$middleware中添加'Fideloper\Proxy\TrustProxies',

```php
protected $middleware = [
	//.....
        'Fideloper\Proxy\TrustProxies',
```

3、设置受信任的反向代理
操作config/trustedproxy.php

```php
<?php

return [
    'proxies' => [
        '192.168.10.10', //受信任的反向代理服务器地址
    ],

// 默认代理配置
'headers' => [
    (defined('Illuminate\Http\Request::HEADER_FORWARDED') ? Illuminate\Http\Request::HEADER_FORWARDED : 'forwarded') => 'FORWARDED',
    \Illuminate\Http\Request::HEADER_CLIENT_IP    => 'X_FORWARDED_FOR',
    \Illuminate\Http\Request::HEADER_CLIENT_HOST  => 'X_FORWARDED_HOST', //AWS Elastic Load Balancing或Heroku则为null
    \Illuminate\Http\Request::HEADER_CLIENT_PROTO => 'X_FORWARDED_PROTO',
    \Illuminate\Http\Request::HEADER_CLIENT_PORT  => 'X_FORWARDED_PORT',
]

];
```

重启laravel服务器httpd，完成