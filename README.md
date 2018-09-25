# Laravel Notes
These notes relate to creating a Q&A system.  

## Initialising a project
1. `laravel new <projectName>`
2. `cd <projectName>`
3. Set-up database
4. Update `.env` with database name
5. `php artisan make:auth`  


## Schema & Migrations
`php artisan make:model <ModelName> -m`

Example schema:  
```javascript
$table->string('title');  
$table->string('slug')->unique();  
$table->text('body');  
$table->unsignedInteger('views')->default(0);  
$table->integer('votes')->default(0);  
$table->unsignedInteger('best_answer_id')->nullable();  
$table->timestamp('created_at')->default(DB::raw('CURRENT_TIMESTAMP'));  
$table->timestamp('updated_at')->default(DB::raw('CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP'));
```

### Setting foreign key  
```javascript
$table->foreign('user_id')->references('id')->on('users')->onDelete('cascade');
```  
This sets a foreign key for 'user_id' to the 'id' in the 'users' table. It also auto-removes all rows related to the user if their record is deleted from the users table.  

`php artisan migrate`  

## Setting relationship
In `Question.php`:  
```php
protected $fillable = ['title', 'body'];

/**
 * Questions belong to users:
 */
public function user() {
    return $this->belongsTo(User::class);
}
```

In `User.php`:  
```php
/**
 * Users may have many questions:
 */
public function questions()
{
    return $this->hasMany(Question::class);
}
```

## Make a Controller with a Resource and Model  
In `web.php`:
```php
Route::resource('questions', 'QuestionsController');
```

_NOTE:_
You could also do the following to be more specific:
```php
Route::get('/dummy', 'DummyController@index');
Route::post('/dummy', 'DummyController@store');
Route::delete('/dummy', 'DummyController@destroy');
:
:
```
...then
```php
php artisan make:controller QuestionsController --resource --model=Question
```
The `--resource` flag adds the following to the controller:  

| Flag      | Description |
|:----------|:------------|
| index()   | Display a listing of the resource. |
| create()  | Show the form for creating a new resource. |
| store()   | Store a newly created resource in storage. |
| show()    | Display the specified resource. |
| edit()    | Show the form for editing the specified resource. |
| update()  | Update the specified resource in storage. |
| destroy() | Remove the specified resource from storage. |

The `--model=Question` flag indicates which model to use.
## Eloquent
In `QuestionsController.php`  
Get all questions but show only the latest 5 initially:
```php
$questions = Question::latest()->paginate(5);
```

## Pass data to a view
In `QuestionsController.php`  
```php
return view('questions.index', compact('questions'));
```

## String Manipulation
```php
// Limit string length:
str_limit( $original, 100 );
```

## Views
```php
// Show pagination links:
{{ $questions->links() }}
```

## Override Laravel items
`php artisan vendor:publish --tag=laravel-pagination`  

## Defining attributes that aren't in Models
In `index.blade.php` we can specify `$question->url`, `$question->user->url` and `$question->created_dateß`:
```php
<div class="media-body">
    <h3 class="mt-0"><a href="{{ $question->url }}">{{ $question->title }}</a></h3>
    <p class="lead">
        Asked by <a href="{{ $question->user->url }}">{{ $question->user->name }}</a>
        <small class="text-muted">{{ $question->created_date }}</small>
    </p>
    {{ str_limit($question->body, 100) }}
</div>
```
We therefore need to create the attribute accssors in the `Question.php` model:
```php
/**
 * Attribute accessor for $question->url required in the view:
 */
public function getUrlAttribute()
{
    return route('questions.show', $this->id);
}

/**
 * Accessor for $question->created_date:
 */
public function getCreatedDateAttribute()
{
    return $this->created_at->diffForHumans();
}
```
...and in `User.php`:
```php
/**
 * Attribute accessor for $question->user->url required in the view:
 */
public function getUrlAttribute()
{
    // return route('questions.show', $this->id);
    return '#';
}
```
### Dynamic class names via accessors
If we have:
```html
<div class="status {{ $question->status }}">
```
then we can create an accessor for the class setting:
```php
/**
 * Accessor for $question->status:
 *
 * Returns a class name relevant to the answered status of the question.
 */
public function getStatusAttribute()
{
    if ($this->answers > 0) {
        if($this->best_answer_id) {
            return 'answered-accepted';
        }
        return 'answered';
    }
    return 'unanswered';
}
```

## Tinker examples
```tinker
>>> $q = App\Question::first();

>>> $q->created_at;

>>> $q->created_at->diffForHumans();

>>> $q->created_at->format('Y-m-d H:i');
```

## Add Laravel debug bar
`composer require barryvdh/laravel-debugbar --dev`

## Eager loading from database
Lazy loading is where multiple queries are used to extract database data:
```php
$questions = Question::latest()->paginate(5);
```
The above results in 5 calls to get user data! This can be overcome by adding a `with()` clause that makes use of the model relationships we created earlier:
```php
$questions = Question::with('user')->latest()->paginate(5);
```
_This line is found in `QuestionsController.php`_
