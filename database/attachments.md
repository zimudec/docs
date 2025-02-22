# Database: File Attachments

## Introduction

Winter CMS provides a few different ways to manage files depending on your needs. File attachments are files that are stored on the filesystem and have database records associated with them in order to simplify connecting them to other database records.

When using the `System\Models\File` model, you are able to configure the disk and paths that are used to store and retrieve the files managed by that model by modifying the `storage.uploads` setting in the [`config/cms.php` file](https://github.com/wintercms/winter/blob/develop/config/cms.php#L317).

## File attachments

Models can support file attachments using a subset of the [polymorphic relationship](../database/relations#polymorphic-relations). The `$attachOne` or `$attachMany` relations are designed for linking a file to a database record called "attachments". In almost all cases the `System\Models\File` model is used to safekeep this relationship where reference to the files are stored as records in the `system_files` table and have a polymorphic relation to the parent model.

In the examples below the model has a single Avatar attachment model and many Photo attachment models.

A single file attachment:

```php
public $attachOne = [
    'avatar' => 'System\Models\File'
];
```

Multiple file attachments:

```php
public $attachMany = [
    'photos' => 'System\Models\File'
];
```

> **NOTE:** If you have a column in your model's table with the same name as the attachment relationship it will not work. Attachments and the FileUpload FormWidget work using relationships, so if there is a column with the same name present in the table itself it will cause issues.

Protected attachments are uploaded to the File Upload disk's **uploads/protected** directory which is not accessible for the direct access from the Web. A protected file attachment is defined by setting the *public* argument to `false`:

```php
public $attachOne = [
    'avatar' => ['System\Models\File', 'public' => false]
];
```

### Creating new attachments

For singular attach relations (`$attachOne`), you may create an attachment directly via the model relationship, by setting its value using the `Input::file` method, which reads the file data from an input upload.

```php
$model->avatar = Input::file('file_input');
```

You may also pass a string to the `data` attribute that contains an absolute path to a local file.

```php
$model->avatar = '/path/to/somefile.jpg';
```

Sometimes it may also be useful to create a `File` instance directly from (raw) data:

```php
$file = (new System\Models\File)->fromData('Some content', 'sometext.txt');
```

For multiple attach relations (`$attachMany`), you may use the `create` method on the relationship instead, notice the file object is associated to the `data` attribute. This approach can be used for singular relations too, if you prefer.

```php
$model->avatar()->create(['data' => Input::file('file_input')]);
```

Alternatively, you can prepare a File model before hand, then manually associate the relationship later. Notice the `is_public` attribute must be set explicitly using this approach.

```php
$file = new System\Models\File;
$file->data = Input::file('file_input');
$file->is_public = true;
$file->save();

$model->avatar()->add($file);
```

You can also add a file from a URL. To work this method, you need install cURL PHP Extension.

```php
$file = new System\Models\File;
$file->fromUrl('https://example.com/uploads/public/path/to/avatar.jpg');

$user->avatar()->add($file);
```

Occasionally you may need to change a file name. You may do so by using second method parameter.

```php
    $file->fromUrl('https://example.com/uploads/public/path/to/avatar.jpg', 'somefilename.jpg');
```

### Viewing attachments

The `getPath` method returns the full URL of an uploaded public file. The following code would print something like **example.com/uploads/public/path/to/avatar.jpg**

```php
echo $model->avatar->getPath();
```

Returning multiple attachment file paths:

```php
foreach ($model->photos as $photo) {
    echo $photo->getPath();
}
```

The `getLocalPath` method will return an absolute path of an uploaded file in the local filesystem.

```php
echo $model->avatar->getLocalPath();
```

To output the file contents directly, use the `output` method, this will include the necessary headers for downloading the file:

```php
echo $model->avatar->output();
```

You can resize an image with the `getThumb` method. The method takes 3 parameters - image width, image height and the options parameter. Read more about these parameters on the [Image Resizing](../services/image-resizing#available-parameters) page.

### Usage example

This section shows a full usage example of the model attachments feature - from defining the relation in a model to displaying the uploaded image on a page.

Inside your model define a relationship to the `System\Models\File` class, for example:

```php
class Post extends Model
{
    public $attachOne = [
        'featured_image' => 'System\Models\File'
    ];
}
```

Build a form for uploading a file:

```html
<?= Form::open(['files' => true]) ?>

    <input name="example_file" type="file">

    <button type="submit">Upload File</button>

<?= Form::close() ?>
```

Process the uploaded file on the server and attach it to a model:

```php
// Find the Blog Post model
$post = Post::find(1);

// Save the featured image of the Blog Post model
if (Input::hasFile('example_file')) {
    $post->featured_image = Input::file('example_file');
}
```

Alternatively, you can use [deferred binding](../database/relations#deferred-binding) to defer the relationship:

```php
// Find the Blog Post model
$post = Post::find(1);

// Look for the postback data 'example_file' in the HTML form above
$fileFromPost = Input::file('example_file');

// If it exists, save it as the featured image with a deferred session key
if ($fileFromPost) {
    $post->featured_image()->create(['data' => $fileFromPost], $sessionKey);
}
```

Display the uploaded file on a page:

```php
// Find the Blog Post model again
$post = Post::find(1);

// Look for the featured image address, otherwise use a default one
if ($post->featured_image) {
    $featuredImage = $post->featured_image->getPath();
}
else {
    $featuredImage = 'http://placehold.it/220x300';
}

<img src="<?= $featuredImage ?>" alt="Featured Image">
```

If you need to access the owner of a file, you can use the `attachment` property of the `File` model:

```php
public $morphTo = [
    'attachment' => []
];
```

Example:

```php
$user = $file->attachment;
```

For more information read the [polymorphic relationships](../database/relations#polymorphic-relations)

### Validation example

The example below uses [array validation](../services/validation#validating-arrays) to validate `$attachMany` relationships.

```php
use Winter\Storm\Database\Traits\Validation;
use System\Models\File;
use Model;

class Gallery extends Model
{
    use Validation;

    public $attachMany = [
        'photos' => File::class
    ];

    public $rules = [
        'photos'   => 'required',
        'photos.*' => 'image|max:1000|dimensions:min_width=100,min_height=100'
    ];

    /* some other code */
}
```

For more information on the `attribute.*` syntax used above, see [validating arrays](../services/validation#validating-arrays).
