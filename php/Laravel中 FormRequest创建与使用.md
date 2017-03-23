## Laravel 中 FormRequest 创建与使用

在开发中总会有很多 CRUD 操作. 而往往也要对数据进行验证/过滤. Laravel 已经提供了很强大的验证服务了,  无论是在控制器还是模型中都能很方便的使用.

但是, 对于用于验证数据的验证规则, 却要总是去反复去书写. 根据DRY原则(Don't repeat yourself) 应该尽量减少重复的代码. 所以用FormRequest来做表单验证还是可以的.



#### 创建文件

```bash
php artisan make:controller TestController	#控制器
php artisan make:request TestRequest		#FormRequest
```



#### 路由

```php
Route::match(['get', 'post'], '/', 'TestController@request');
```



#### 控制器

```php
# app/Http/Controllers/TestController.php

namespace App\Http\Controllers;

use App\Http\Requests\TestRequest;
use App\Http\Controllers\Controller;

class TestController extends Controller
{
	public function request(TestRequest $request)
	{
		return '验证通过...';
	}
}
```



#### FormRequest

```php
# app/Http/Requests/TestRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class TestRequest extends FormRequest
{
	// 是否允许请求
    public function authorize()
    {
        return true;
    }

	// 验证规则
    public function rules()
    {
        return [
			'name'		=>	'required|min:1|max:20'
		];
    }

	// 错误信息
	public function messages()
	{
		return [
			'name.required'		=>	'名称不能为空',
			'name.min'			=>	'名称最短不能小于1位',
			'name.max'			=>	'名称最长不能大于20位'
		];
	}

	// 重写父类中的方法, 用于自定义响应内容, 可选
	public function response(array $errors)
	{
		return parent::response([
			'status'	=>	0,
			'msg'		=>	current(current($errors))
		]);
	}

	// 重写父类中的方法, 用于自定义响应状态, 可选
	public function forbiddenResponse()
	{
		return parent::forbiddenResponse();
	}
}
```
