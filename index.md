# Quickstart for a Larvel project with Elasticsearch

This quickstart assumes, that apache, php, mysql, composer and elasticsearch are installed and ready on the target machine.

Create your project:

```
composer create-project laravel/laravel <project-name>
```

Set your database options in the `.env` file.

```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=
```

MySQL may sometimes complain about keys being too long. If so, change this setting in `app/Providers/AppServiceProvider.php`:

```
use Illuminate\Support\Facades\Schema;

public function boot(){

    Schema::defaultStringLength(191);

}
```

Make sure your database connection is working:

```
php artisan migrate
```

#
Install Laravel Breeze to enable authentication:

```
composer require laravel/breeze --dev &&
php artisan breeze:install --dark &&
php artisan migrate &&
npm install
```

Create your admin user:

```
php artisan ti

User::create([
    "name" => "<your-name>",
    "email" => "<your-email>",
    "password" => bcrypt("<your-password>")
])
```

#

Create a demo model:

```
php artisan make:model -mf Post
```

Inside `database/migrations/create_posts_table.php`, set the migration data:

```
public function up(){

    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->text('title');
        $table->text('content');
        $table->timestamps();
    });

}
```
A factory to create dummy data will also be helpful, therefore make the `title` and `content` columns fillable in your `app/Models/Post.php` model...

```
class Post extends Model{

    use HasFactory;

    protected $fillable = ['title', 'content'];

}
```

...and define the factory in `database/factories/PostFactory.php`:

```
class PostFactory extends Factory{
    
    public function definition(){

        return [
            'title' => fake()->text(32),
            'content' => fake()->text(256)
        ];

    }

}
```

#

Install Laravel Scout to make your models searchable:

```
composer require laravel/scout &&
php artisan vendor:publish --provider="Laravel\Scout\ScoutServiceProvider"
```

Install `jeroen-g/explorer` to connect Laravel Scout to Elasticsearch

```
composer require jeroen-g/explorer &&
php artisan vendor:publish --tag=explorer.config
```

Then, inside your `config/scout.php`, set the default search engine to `elastic`:

```
'driver' => env('SCOUT_DRIVER', 'elastic'),
```

#

Add your `Post` model to the list of searchable indexes in `config/explorer.php`:

```
'indexes' => [
    \App\Models\Post::class
],
```

Add the `Searchable` trait to your `Post` model and implement the `Explored` interface in `app/Models/Post.php` so it looks like this:

```
namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use JeroenG\Explorer\Application\Explored;
use Laravel\Scout\Searchable;

class Post extends Model implements Explored{

    use HasFactory;
    use Searchable;

    protected $fillable = ['title', 'content'];

    public function mappableAs(): array{

        return [
            'id' => 'keyword',
            'title' => 'text',
            'description' => 'text',
            'created_at' => 'date'
        ];

    }

    public function searchableAs(){

        return 'posts';

    }

}
```

Add the `posts` index to your Elasticsearch cluster:

```
curl -XPUT localhost:9200/posts '
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  },
  "mappings": {
    "properties": {
      "id": { "type": "unsigned_long" },
      "title": { "type": "text" },
      "content": { "type": "text" },
      "created_at": { "type": "date" }
    }
  }
}'
```

#

Create a bunch of dummy posts:

```
php artisan ti

Post::factory()->count(10)->create()
```

Test your search:

```
php artisan elastic:search "App\Models\Post" lorem
```

This should return all records that where title or content contains the **word** "lorem"