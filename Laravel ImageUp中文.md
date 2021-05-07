## Laravel ImageUp

`qcod/laravel-imageup` 是一个可让您通过大量自定义功能自动上传，调整图像大小和裁剪图像功能。

### 安装

您可以通过composer安装该软件包：

```bash
$ composer require qcod/laravel-imageup
```

软件包将自动注册。如果需要手动注册，可以通过将其添加到`config/app.php`providers数组中来进行注册：

```php
QCod\ImageUp\ImageUpServiceProvider::class
```

您可以选择使用以下方式发布配置文件：

```bash
php artisan vendor:publish --provider="QCod\ImageUp\ImageUpServiceProvider" --tag="config"
```

它将[`config/imageup.php`](https://github.com/qcod/laravel-imageup#config-file)使用所有设置进行创建。

### 入门

您只需添加`use HasImageUploads`Eloquent模型并定义所有需要在数据库中存储图像的字段。

**Model**
```php
<?php
namespace App;

use QCod\ImageUp\HasImageUploads;
use Illuminate\Database\Eloquent\Model;

class User extends Model {
    use HasImageUploads;
    
    //假设用户表有'cover'，'avatar'列
    //将所有列标记为
    protected static $imageFields = [
        'cover', 'avatar'
    ];
}
```

标记了模型中的所有图像字段后，只要您通过挂接`Model::saved()`事件保存模型，它将自动上载图像。它还将使用新存储的文件路径更新数据库，如果找到旧图像，则在上载新图像后将其删除。

> 数据库表中的图像字段应与`VARCHAR`，以存储上载图像的路径。

**在控制器中**

```php
<?php
namespace App;
use App\Http\Controllers\Controller;

class UserController extends Controller {
    public function store(Request $request){
        return User::create($request->all());
    }
}
```
> 确保运行`php artisan storage:link`来使用公共存储磁盘中的图像

通过上述设置，当您使用发布请求命中存储方法时，如果request（）上存在`cover`或`avatar`命名的文件，它将被自动上传。

## 上传选项

ImageUp为您提供了大量自定义功能，可根据定义的字段选项处理上传和调整大小的方式，以下是您可以自定义的内容：

```php
<?php
namespace App;

use QCod\ImageUp\HasImageUploads;
use Illuminate\Database\Eloquent\Model;

class User extends Model {
    
    use HasImageUploads;
    
    //选择哪一个磁盘用于上传，可以覆盖 
    protected $imagesUploadDisk = 'local';
    
    //在盘使用上传路径，可以覆盖 
    protected $imagesUploadPath = 'uploads';
    
    //自动上传允许
    protected $autoUploadImages = true;
    
    // 所有选择的图像上传字段
    protected static $imageFields = [
        'avatar' => [
            // 上传后要调整图像大小
            'width' => 200,
            
            // 上传后调整图片大小的高度
            'height' => 100,
            
            // 将true设置为具有给定宽度/高度的裁剪图像，还可以为裁剪传递arr [x，y]坐标。
            'crop' => true,
            
            // 您要上传的磁盘，默认config（'imageup.upload_disk'）
            'disk' => 'public',
            
            // 上传磁盘上的文件夹路径，默认为config（'imageup.upload_directory'）
            'path' => 'avatars',
            
            // 如果图像字段为空，则为占位符图像
            'placeholder' => '/images/avatar-placeholder.svg',
            
            // 上传图片时的图片验证规则
            'rules' => 'image|max:2000',
            
            // 是否覆盖全局自动上传设置（“imageup.auto_upload_images”）
            'auto_upload' => false,
            
            // 如果请求文件的名称不同，则默认值为字段名称
            'file_input' => 'photo',
            
            // 在文件保存之前触发的钩子
            'before_save' => BlurFilter::class,
            
            // 保存文件后触发的钩子
            'after_save' => CreateWatermarkImage::class
        ],
        'cover' => [
            //...    
        ]
    ];
    
    // any other than image file type for upload
    protected static $fileFields = [
            'resume' => [
                // 您要上传的磁盘，默认config（'imageup.upload_disk'）
                'disk' => 'public',
                
                // 上传磁盘上的文件夹路径，默认为config（'imageup.upload_directory'）
                'path' => 'docs',
                
                // 上传图片时的图片验证规则
                'rules' => 'mimes:doc,pdf,docx|max:1000',
                
                // 覆盖全局自动上传设置（“imageup.auto_upload_images”）
                'auto_upload' => false,
                
                // 如果请求文件的名称不同，则默认值为字段名称
                'file_input' => 'cv',
                
                // 在文件保存之前触发的钩子
                'before_save' => HookForBeforeSave::class,
                
                // 保存文件后触发的钩子
                'after_save' => HookForAfterSave::class
            ],
            'cover_letter' => [
                //...    
            ]
        ];
}
```
### 自定义文件名

在某些情况下，您将需要自定义保存的文件名。默认情况下，它将`$file->hashName()`生成哈希值。

您可以通过在模型上添加具有`{fieldName}UploadFilePath`命名约定的方法来做到这一点：

```php
class User extends Model {
    use HasImageUploads;
    
    // assuming `users` table has 'cover', 'avatar' columns
    // mark all the columns as image fields 
    protected static $imageFields = [
        'cover', 'avatar'
    ];
    
    // override cover file name
    protected function coverUploadFilePath($file) {
        return $this->id . '-cover-image.jpg';
    }
}
```

上方将始终将上传的封面图片另存为`uploads/1-cover-image.jpg`。	

> 确保仅从覆盖方法返回相对路径。

请求文件将通过`$file`此方法作为参数传递，因此您可以获取扩展名或原始文件名等来构建文件名。

```php
    // override cover file name
    protected function coverUploadFilePath($file) {
        return $this->id .'-'. $file->getClientOriginalName();
    }
    
    /** Some of methods on file */
    // $file->getClientOriginalExtension()
    // $file->getRealPath()
    // $file->getSize()
    // $file->getMimeType()
```

## 可用方法

您不仅能使用自动上传图像功能。还为您提供以下方法，您可以使用这些方法来手动上载图像并调整其大小。

**注意：**请确保已通过设置`protected $autoUploadImages = false;` 模型或通过调用动态禁用了自动上传`$model->disableAutoUpload()`。您也可以通过调用来`$model->setImagesField(['cover' => ['auto_upload' => false]);` 将其禁用以用于指定字段，否则您将看不到您的手动上传，因为在保存模型时会被自动上传覆盖。

#### $ model-> uploadImage（$ imageFile，$ field = null）/ $ model-> uploadFile（$ docFile，$ field = null）

为给定的$ field上传图像/文件，如果$ field为null，它将上传到数组中定义的第一个图像/文件选项。

```php
$user = User::findOrFail($id);
$user->uploadImage(request()->file('cover'), 'cover');
$user->uploadFile(request()->file('resume'), 'resume');
```

#### $model->setImagesField($fieldsOptions) / $model->setFilesField($fieldsOptions) 

您还可以通过`$model->setImagesField($fieldsOptions) / $model->setFilesField($fieldsOptions)`使用字段选项进行调用来动态设置图像/文件字段，它将替换在模型属性上定义的字段。

```php
$user = User::findOrFail($id);

$fieldOptions = [
    'cover' => [ 'width' => 1000 ],
    'avatar' => [ 'width' => 120, 'crop' => true ],    
];

// 放大图片
$user->setImagesField($fieldOptions);

$fileFieldOption = [
    'resume' => ['path' => 'resumes']
];

// 修改上传磁盘上的文件夹路径
$user->setFilesField($fileFieldOption);
```

#### $model->hasImageField($field) / $model->hasFileField($field)

检查字段是否定义为图像/文件字段。

#### $model->deleteImage($filePath) / $model->deleteFile($filePath) 

删除任何图像/文件（如果存在）。

#### $model->resizeImage($imageFile, $fieldOptions)

如果已经有图像，则可以使用与图像字段相同的选项来调用此方法以调整其大小。

```php
$user = User::findOrFail($id);

//调整图片大小，它将为您提供调整大小的图片，您需要保存它
$imageFile = '/images/some-big-image.jpg';
$image = $user->resizeImage($imageFile, [ 'width' => 120, 'crop' => true ]);

// 您可以使用上传的文件
$imageFile = request()->file('avatar');
$image = $user->resizeImage($imageFile, [ 'width' => 120, 'crop' => true ]);
```

#### $model->cropTo($x, $y)->resizeImage($imageFile, $field = null)

您可以使用此`cropTo()`方法设置裁剪的x和y坐标。如果要从某种字体结束的图像裁剪库中获取坐标，这将非常有用。

```php
$user = User::findOrFail($id);

// 请求文件
$imageFile = request()->file('avatar');

// 请求文件并剪裁到坐标
$coords = request()->only(['crop_x', 'crop_y']);

// 修改文件宽度
$image = $user->cropTo($coords)
    ->resizeImage($imageFile, [ 'width' => 120, 'crop' => true ]);

// 覆盖剪裁到配置
$user->cropTo($coords)
    ->uploadImage(request()->file('cover'), 'avatar');
```

#### $model->imageUrl($field) / $model->fileUrl($field)

提供给定图像/文件字段的上传文件网址。

```php
$user = User::findOrFail($id);

// 在你的视图中
<img src="{{ $user->imageUrl('cover') }}" alt="" />
// http://www.example.com/storage/uploads/iGqUEbCPTv7EuqkndE34CNitlJbFhuxEWmgN9JIh.jpeg
```

#### $model->imageTag($field, $attribute = '')

它为您`<img />`提供了字段的标签。

```html
{!! $model->imageTag('avatar') !!}
<!-- <img src="http://www.example.com/storage/uploads/iGqUEbCPTv7EuqkndE34CNitlJbFhuxEWmgN9JIh.jpeg" /> -->

{!! $model->imageTag('avatar', 'class="float-left mr-3"') !!}
<!-- <img src="http://www.example.com/storage/uploads/iGqUEbCPTv7EuqkndE34CNitlJbFhuxEWmgN9JIh.jpeg" class="float-left mr-3 /> -->
```

### 钩子

挂钩允许您在保存图像之前或之后进行的任何其他逻辑。

##### 定义类型

您可以通过指定类名来定义钩子

```php
protected static $imageFields = [
    'avatar' => [
        'before_save' => BlurFilter::class,
    ],
    'cover' => [
        //...    
    ]
];
```

挂钩类必须具有一个名为的方法`handle`，该方法将在触发挂钩时被调用。干预图像的实例将传递给该`handle`方法。

```php
class BlurFilter {
    public function handle($image) {
        $image->blur(10); //模糊10,具体支持ide自动出来自己看吧
    }
}
```

通过laravel ioc容器解析基于类的钩子，该容器允许您通过构造函数注入任何依赖项。

> 请注意，如果您使用或定义了字段选项，则将在其中调整大小的图像`before`并`after`保存钩子处理程序。当然，您可以随时获取原始图像。`width``height``request()->file('avatar')`

定义第二种类型是回调挂钩。
```php
$user->setImagesField([
    'avatar' => [
        'before_save' => function($image) {
            $image->blur(10);
        },
    ],
    'cover' => [
        //...    
    ]
]);
```

回调还将接收干预图像实例参数。

##### 钩子类型

钩子a`before_save`（图片存储前）和`after_save`（图片存储后）钩子有两种类型。

`before_save`在将映像保存到磁盘之前，将调用该挂钩。对挂钩内的干预所做的任何更改都将应用于保存的图像。

```php
$user->setImagesField([
    'avatar' => [
        'width' => 100,
        'height' => 100,
        'before_save' => function($image) {
            // 修改图片大小
            $image->resize(50, 50);
        },
    ]
]);
```

`after_save`在将映像保存到磁盘之后立即调用该挂钩。

```php
$user->setImagesField([
    'logo' => [
        'after_save' => function($image) {
            // 创建水印
        },
    ]
]);
```

### 配置文件

```php
<?php

return [

    /**
     * 保存磁盘
     */
    'upload_disk' => 'public',

    /**
     * 保存目录
     */
    'upload_directory' => 'uploads',

    /**
     * 是否自动上传
     */
    'auto_upload_images' => true,

    /**
     * 数据库中记录被删除，自动删除图片
     */
    'auto_delete_images' => true,

    /**
     * 设置图像质量
     */
    'resize_image_quality' => 80
];
```

下载地址

https://github.com/qcod/laravel-imageup