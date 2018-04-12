## Laravel 中 FormRequest 创建与使用

在开发中会有很多 CRUD 操作，往往也会需要对数据进行 验证、过滤。Laravel 已经提供了很强大的验证服务了,  无论是在控制器还是模型中都能很方便的使用。

但是，对于用于验证数据的规则，却要总是去反复去书写。根据DRY原则（Don't repeat yourself） 应该尽量减少重复的代码。那么使用 FormRequest 可以尽量避免这种情况。



## 01. 创建文件

```bash
php artisan make:controller TestController  # 控制器
php artisan make:request TestRequest        # FormRequest
```



## 02. 添加路由

```php
Route::match(['get', 'post'], '/', 'TestController@request');
```



## 03. 控制器

```php
# app/Http/Controllers/TestController.php

namespace App\Http\Controllers;

use App\Http\Requests\TestRequest;
use App\Http\Controllers\Controller;

class TestController extends Controller
{
    public function request(TestRequest $request)
    {
        return '验证通过…';
    }
}
```



## 04. FormRequest

```php
# app/Http/Requests/TestRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class TestRequest extends FormRequest
{
    // 是否授权请求
    public function authorize()
    {
        return true;
    }

    // 验证规则
    public function rules()
    {
        return [
            'name' => 'required|min:1|max:20'
        ];
    }

    // 错误信息
    public function messages()
    {
        return [
            'name.required' => '名称不能为空',
            'name.min'      => '名称最短不能小于 1 位',
            'name.max'      => '名称最长不能大于 20 位'
        ];
    }

    // 可选，重写父类中的方法，用于自定义响应内容
    public function response(array $errors)
    {
        return parent::response([
            'status' => 0,
            'msg'    => current(current($errors))
        ]);
    }

    // 可选，重写父类中的方法，用于自定义响应状态
    public function forbiddenResponse()
    {
        return parent::forbiddenResponse();
    }
}
```

