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

## Inline routing
```html
<a href="{{ route('questions.create') }}" class="btn btn-outline-secondary">Ask Question</a>
```
```html
<form action="{{ route('questions.store') }}" method="post" class="form">
```

## Forms
### Error feedback
```html
@csrf

<input type="text" class="form-control {{ $errors->has('title') ? 'is-invalid' : '' }}" name="title" id="title" aria-describedby="helptitle" placeholder="Your question here">

@if ( $errors->has('title') )
    <div class="invalid-feedback">
        <strong>{{ $errors->first('title') }}</strong>
    </div>
@else
    <small id="helptitle" class="form-text text-muted">Keep it short!</small>
@endif
```

## Validation
We'll use a seperate file for the validation of form submissions (requests), so:
`php artisan make:request AskQuestionRequest`
Now open `app\Http\Requests\AskQuestionRequest.php` to add validations/rules.

```php
public function rules()
{
    return [
        'title' => 'required|min=5|max=255',
        'body' =>'required|min=5'
    ];
}
```
In `QuestionsController.php` modify the `store()` function:
```php
public function store(AskQuestionRequest $request)
{
    // Get current user via questions relationship:
    $request->user()->questions()->create( $request->only('title', 'body') );
    //                  ^ adds user_id

    return redirect()->route('questions.index')->with('success', "Your question has been submitted.");
}
```
...and import the `AskQuestionRequest` file:
```php
use App\Http\Requests\AskQuestionRequest;
```

## Editing
Add 'edit' button to right of record's title:
```html
<div class="d-flex align-items-center">
    <h3 class="mt-0"><a href="{{ $question->url }}">{{ $question->title }}</a></h3>
    <div class="ml-auto">
        <a href="{{ route('questions.edit', $question->id) }}" class="btn btn-sm btn-outline-info mr-1">Edit</a>
    </div>
</div>
```
_Note_ the `{{ route('questions.edit', $question->id) }}` in the href.

### Showing old/editable values
`
{{ old('title', $question->title) }}
`

### Save edited data
In `QuestionsController.php` modify the `update()` function:
```php
public function update(AskQuestionRequest $request, Question $question)
{
    $question->update( $request->only(['title', 'body']) );
    return redirect(route('questions.index'))->with('success', 'Your question has been updated.');
}
```
_Note_ the `update(AskQuestionRequest...` parameter.  

### Delete a record
In `QuestionsController.php`:
```php
public function destroy(Question $question)
{
    $question->delete();
    return redirect(route('questions.index'))->with('success', "That question has now been deeted.");
}
```

## Escaped output with new-lines too
`{!! nl2br(e($question->body)) !!}`

## Override controller method
In `web.php`:
```php
// Let a resource handle all routes except 'show':
Route::resource('questions', 'QuestionsController')->except('show');

// Show question details using the slug - instead of the (default) question id:
Route::get('/questiosns/{slug}', 'QuestionsController@show')->name('questions.show');
```
In `RouteServiceProvider.php`:
```php
public function boot()
{
    Route::bind('slug', function($slug) {
        // $question = Question::where('slug', $slug)->first();
        // return $question ? $question : abort(404);
        // ...or:
        return Question::where('slug', $slug)->first() ?? abort(404);
    });

    parent::boot();
}
```
