# On Page ® PHP library

With this library you can easy query your data using an On Page ® API token.

## Installation

```
composer config repositories.onpage vcs 'https://github.com/onpage-dev/onpage-php.git'
composer require onpage-dev/onpage-php:^v1.1
```

To update to the latest version, you can launch

```
composer require onpage-dev/onpage-php:^v1.1
```

Of course, remember to include the composer autoload:

```php
<?php
require 'vendor/autoload.php';
```

## Usage

### Setup

```php
$api = new \OnPage\Api('acme-inc', $api_token);
```

### Get structure information

```php
// Retrieve info about the schema:
echo $api->schema->label;

// Retrieve a resource given its name or ID
$res = $api->schema->resource('products');
foreach ($res->fields() as $field) {
    echo "$field->getLabel()\n"; // Main image
    echo "$field->getLabel('zh')\n"; // "Main Image" but in Chinese
    echo "$field->name\n"; // "main_image"
    echo "$field->type\n"; // string|file|image|real|int|text|...
    echo "$field->unit\n"; // null|kg|...
    echo "$field->is_multiple\n"; // true|false
    echo "$field->is_translatable\n"; // true|false
}
```

### Query your data

```php
// Retrieve all records of a resource (returns a laravel collection of \OnPage\Thing)
$products = $api->schema->query('products')->all();
foreach ($products as $prod) {
    // ...
}

// Get only the first item
$prod = $api->query('products')->first();
```

### Filters and deletions

```php
// Retrieve all records of a resource (returns a laravel collection of \OnPage\Thing)
$api->query('products')
    ->where('_id', 42) // = is the default operator
    ->first();

// Other filters
$api->query('products')
    ->where('name', 'like', 'shoes') // you can specify a different operator
    ->where('category.name', 'Nike') // you can query relations
    ->where('dimension', '>', 10) // you get it
    ->all(); // returns a collection with all your records


// You can just smply delete data the same way:
$api->query('products')
    ->where(...)
    ->delete();
```

### Get thing values

Use the val() function to get the first value in a field.
Use the values() function to get all values in a field as a collection.

```php
$cat = $api->query('categories')->first();
echo $cat->val('name');
echo $cat->val('dimension');
echo $cat->val('description', 'zh'); // you can specify a language

// Or set the default language
$api->schema->lang = 'zh';
echo $cat->val('name'); // 再见

// The values function is useful for multivalue fields, it will return a laravel collection of values.
echo $cat->values('bullet_points')->implode('; ');
```

#### Files

For `image` and `file` fields, the returned value will be an instance of `\OnPage\File::class`.
To get a file or image url use the `->link()` function. The link will point to the original file.

```php
# original size
$product->val('specsheet')->name // icecream-spec.pdf
$product->val('specsheet')->token // R417C0YAM90RF
$product->val('specsheet')->link() // https://acme-inc.onpage.it/api/storage/R417C0YAM90RF?name=icecream-spec.pdf
```

To turn images into a thumbnail add an array of options as shown below:

```php
# maintain proportions width 200px
$product->val('cover_image')->link(['x' => 200])

# maintain proportions height 100px
$product->val('cover_image')->link(['y' => 100])

# crop image to width 200px and height 100px
$product->val('cover_image')->link(['x' => 200, 'y' => 100])

# maintain proportions and contain in a rectangle of width 200px and height 100px
$product->val('cover_image')->link(['x' => 200, 'y' => 100, 'contain' => true])

# convert the image to png (default is jpg)
$product->val('cover_image')->link(['x' => 200, 'format' => 'png'])
```

#### Other utilities

```php
// Speed things up by only loading some fields
$api->query('products')->loadFields(['title'])->all();

// Get a mapping between two fields or a field and the thing ID
$api->query('products')->map('code');
// [ 'MYSKU100' => 1827, 'MYSKU101' => 1828, ... ]

$api->query('products')->map('code', 'title');
// [ 'MYSKU100' => 'Apples', 'MYSKU101' => 'Bananas', ... ]
```

### Get thing relations

```php
// You need to specify the relations using the "with" method
$cat = $api->query('categories')
    ->with('subcategories')
    ->first();
$subcategories = $cat->rel('subcategories');
foreach ($subcategories as $subcategory) {
    echo $subcategory->val('name');
}

// You can also preload nested subcategories
$cat = $api->query('categories')
    ->with('subcategories.articles.colors')
    ->first();

// Of course you can use it with the ->all() method
$products_with_colors = $api->query('products')
    ->with('colors')
    ->all();
foreach ($products_with_colors as $prod) {
    echo $prod->val('name');
    foreach ($prod->colors as $color) {
        echo $color->val('name');
    }
}
```

# Creating and updating things

To create or update a record, you need to create a Thing Editor.
There are two ways to get a Thing Editor:

1. Using the **Resource Writer**
2. Calling `->editor()` on a `Op\Thing`

## Using the Resource Writer (first method)

This class allows you to edit many records at once.
You can easily obtain the editor calling:

```php
$writer = $api->resource('categories')->writer();
```

Now that you have a **Resource Writer**, you can use it to create things:

```php
$editor = $writer->createThing();
$editor->set('name', 'Element 1');
$editor->setRel('category', [ 12345 ]); // array with category IDs
```

...and to update existing things:

```php
$editor = $writer->updateThing(736251); // The id of the element you want to update
$editor->set('description', 'Element 1 description');
```

Finally, you need to send the request to the On Page server:

```php
// this will create and update all the things as requested above
$writer->save();
```

## Updating a single item (second method)

```php
$product = $api->query('products')->where('name', 'Plastic Duck')->first();

$editor = $product->editor();
$editor->set('description', 'This yellow plastic duck will be your best friend');
$editor->set('description', '这只黄色塑料鸭将是你最好的朋友', 'zh'); // you can specify language

// Save all the edits at once using the save method
$editor->save();

```

## Updating translations

Just add the language code as the third argument to the `set` function:

```php
// Update the value in the default language
$editor->set('description', 'This yellow plastic duck will be your best friend');

// Specify another the language
$editor->set('description', '这只黄色塑料鸭将是你最好的朋友', 'zh');
```

## Updating files

You can upload files to On Page using the FileUpload class:

```php
$editor->set('image', new \OnPage\FileUpload('/path/to/bird.jpg')); // upload file
```

Or you can also upload a file using a public URL:

```php
$editor->set('image', 'https://mysite.com/bird_cover.jpg'); // specify file by url
```

## Updating multivalue fields

For multivalue fields you only need to replace `->set` with `->setValues` and pass an array of values as the second argument:

```php
$editor->setValues('bullet_points', [
    'Durable plastic',
    'Bright yellow color',
    'Compostable'
]);
```

## Updating relations

To update relations, you can use the `->setRel(relation_name, related_ids)`:

```php
$editor->setRel('features', [
    425790,
    547023,
    240289,
]);
```
