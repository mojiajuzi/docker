### Laravel自带的登录注册组件简单使用

在这里我们需要理解Laravle中两个名词
- Guards:定义了对于每一个请求，如何对用户进行认证
- Providers: 定义了从哪里获取用户

####  使用脚手架生成登录注册页面
```bash
php artisan make:auth
```

#### 查看配置`config/auth.php`, 该文件用于配置如何登录,注册,重置密码
##### Providers Config
```php
'providers' => [
    'users' => [
        'driver' => 'eloquent',
        'model' => App\User::class,
    ],

    // 'users' => [
    //     'driver' => 'database',
    //     'table' => 'users',
    // ],
],
```
在Providers中，定义了两个数组,　一个是以`Eloquent`模型作为驱动，并且使用`App\Users`这个数据模型作为存储用户数据的载体，当然也可以使用`database`作为驱动,直接指明`table`作为存储载体

指定了如何获取到用户数据以后，接下来就需要配置如何验证用户

##### Guards Config
```php
'guards' => [
    'web' => [
        'driver' => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver' => 'token',
        'provider' => 'users',
    ],
],
```
在Guards中,配置了两种的验证方式,一种是`web`,通过`session`来验证用户信息,另一种是通过令牌`token`来验证用户，这两种方式默认都使用以`Eloquent`类型的`App\User`模型作为存储用户信息的载体

##### 默认配置
```php
'defaults' => [
    'guard' => 'web',
    'passwords' => 'users',
]
```

#### 路由(`routes\web.php`)
在`web.php`这个路由文件中,可以找到如下的语句
```PHP
Auth::routes();
```
由于`Auth`是一个别名,其定义在`config/app.php`文件的`aliases`数组中，该数组对应的是一个[Facades](https://laravel.com/docs/5.6/facades),`Illuminate\Support\Facades\Auth`,在这个类里面我们找到了`routes`方法

```php
    public static function routes()
    {
        static::$app->make('router')->auth();
    }
```
在这个方法中，从`$app`对象中获取`router`类示例,而这个类示例是在`Illuminate/Foundation/Application.php`文件中的`registerCoreContainerAliases`方法中进行注册的，因此，我们可以找到`Illuminate\Routing\Router`类，在这个类中，我们可以看到

```PHP
public function auth()
{
    // Authentication Routes...
    $this->get('login', 'Auth\LoginController@showLoginForm')->name('login');
    $this->post('login', 'Auth\LoginController@login');
    $this->post('logout', 'Auth\LoginController@logout')->name('logout');

    // Registration Routes...
    $this->get('register', 'Auth\RegisterController@showRegistrationForm')->name('register');
    $this->post('register', 'Auth\RegisterController@register');

    // Password Reset Routes...
    $this->get('password/reset', 'Auth\ForgotPasswordController@showLinkRequestForm')->name('password.request');
    $this->post('password/email', 'Auth\ForgotPasswordController@sendResetLinkEmail')->name('password.email');
    $this->get('password/reset/{token}', 'Auth\ResetPasswordController@showResetForm')->name('password.reset');
    $this->post('password/reset', 'Auth\ResetPasswordController@reset');
}
```
在这里面，找到了我们需要的方法

#### 控制器(Controller)
使用`RegistersUsers`这个trait以及 `App\RegisterController`，实现用户注册的功能
在`register`方法中，做了一下几个动作

```php
public function register(Request $request)
{
    $this->validator($request->all())->validate();

    event(new Registered($user = $this->create($request->all())));

    $this->guard()->login($user);

    return $this->registered($request, $user)
                    ?: redirect($this->redirectPath());
}
```
1. 验证用户提交数据
1. 将用户写入数据库
1. 触发用户注册事件
1. 建立用户会话(session, cookie)
1. 跳转到`/home`页面
