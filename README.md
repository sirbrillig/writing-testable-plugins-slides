				
					
# Writing Testable Plugins
					
---

				
					
## Who am I?

Payton Swick

Code Wrangler at Automattic
(we make WordPress.com)

@sirbrillig
					
---

				
					
### What is a unit test?


Code to test that one piece of a system operates as expected.

					
---

				
					
There are units of different sizes.
					
---

				
					
Database tests are a big unit. Function tests are a small unit. Focus on the small.
					
---

				
					
Smaller systems are easier to understand.
					
---

				
					
It's nice to be able to test a plugin *without* WordPress.
					
---

				
					
You might not need unit tests!</p>
(But it can't hurt!)</p>
					
---

				
					
### What is a unit test? (pt. 2)


Unit tests check the *output* for a given *input*.

					
---

				
					
### What is testable code?

Code that makes it easy to test input and output in isolation.
					
---

				
					
Isolation is the hard part. Most code has dependencies and side effects.
					
---

				
					
Dependencies: this code's *input* is not just in its arguments.
					
---

				
					
To reduce dependencies:


1. Put dependencies in variables that can be overridden.


2. Pass input to your functions rather than having your function fetch input.

					
---

				
					
Nope.</p>
```php
function update_post_status( $post_id ) {
	$post = get_post( $post_id );
	$post->post_status = 'publish';
	return wp_update_post( $post );
}

update_post_status( 123 );
```
Yay!</p>
```php
function update_post_status( $post ) {
	$post->post_status = 'publish';
	return wp_update_post( $post );
}

update_post_status( get_post( 123 ) );
```
					
---

				
					
Nope.</p>
```php
class MyPlugin {
	function update_post_status( $post_id ) {
		$updater = new Updater();
		$updater->update_post( $post_id );
	}
}
```
Yay!</p>
```php
class MyPlugin {
	public $updater;

	public function __construct( $updater = null ) {
		$this->updater = $updater ? $updater : new Updater();
	}

	public function update_post_status( $post_id ) {
		$this->updater->update_post( $post_id );
	}
}
```
					
---

				
					
Allowing dependencies to be overwritten is called "Dependency Injection".

Tests can isolate one part of code from another by replacing the dependencies.

					
---

				
					
Side Effects: this code's *output* is not just in its return value.
					
---

				
					
To reduce side effects:

1. Have each function do just one thing.


2. Make more (specific) classes and functions.

					
---

				
					
Nope.</p>
```php
function update_post_data( $post ) {
	$post->post_status = 'publish';
	$post->post_excerpt = substr( $post->post_content, 0, 140 );
	return wp_update_post( $post );
}

update_post_data( $post );
```
Yay!</p>
```php
function update_post_status( $post ) {
	$post->post_status = 'publish';
	return $post;
}

function update_post_excerpt( $post ) {
	$post->post_excerpt = substr( $post->post_content, 0, 140 );
	return $post;
}

$post = update_post_status( get_post( 123 ) );
$post = update_post_excerpt( $post );
wp_update_post( $post );
```
					
---

				
					
### Payton's Testable</span> Precepts
					
---

				
					
Precept 1:</p>
Always use classes, never global functions.
					
---

				
					
Nope.</p>
```php
function otherpages_add_shortcode() {
...
}
```

Yay!</p>
```php
class Otherpages {
	public function add_shortcode() {
	...
	}
}
```
					
---

				
					
One class to do one thing. Don't be afraid of too many classes.
					
---

				
					
Where two classes interact, we can place a *mock*.
					
---

				
					
Mocks (aka Stubs):
fake objects or functions that behave in a way we can control.
					
---

				
					
A simple mock object is just a `new StdClass()`,
and a simple mock function is just a `function`,
but I'll suggest a library called `Spies`.

https://github.com/sirbrillig/spies

(I wrote it.)
					
---

				
					
Code getting input from a function dependency:
```php
function get_the_content( id ) {
	$post = get_post( id );
	return $post->post_content;
}
```
Test with a mock function dependency:
```php
\Spies\mock_function( 'get_post' )->that_returns(
	(object) [ 'ID' => 123, 'post_content' => 'hello' ]
);
$result = get_the_content( 123 );
$this->assertEquals( 'hello', $result );
```
					
---

				
					
Code getting input from an object dependency:
```php
class MyPlugin {
	public $getter;

	public function __construct( $getter = null ) {
		$this->getter = $getter ? $getter : new Getter();
	}

	public function get_data_content() {
		$post = $this->getter->get_post();
		return $post->post_content;
	}
}
```
Test with a mock object dependency:
```php
$getter = \Spies\mock_object();
$getter->add_method( 'get_post' )->that_returns(
	(object) [ 'ID' => 123, 'post_content' => 'hello' ]
);
$plugin = new MyPlugin( $getter );
$result = $plugin->get_data_content();
$this->assertEquals( 'hello', $result );
```
					
---

				
					
Precept 2:</p>
Don't use static functions except as generators.
					
---

				
					
Why are static methods risky? They cannot easily be mocked.
					
---

				
					
Generators are ok.

```php
class MyPlugin {
	public $getter;

	public function __construct( $getter ) {
		$this->getter = $getter;
	}

	public static function get_instance() {
		return new MyPlugin( new Getter() );
	}
}
```
Generators don't always need to be tested.
					
---

				
					
Static functions that have no dependencies and no side-effects are ok.

```php
class MyPlugin {
	public static function is_published_post( $post ) {
		return $post->post_status === 'publish';
	}
}
```
These can be tested without mocks.
					
---

				
					
Precept 3:</p>
Use instance variables as constants.
					
---

				
					
Instance variables can be changed during runtime to produce different for testing.
					
---

				
					
Nope.</p>
```php
define( 'DATA_DIR', '/data/dir' );

class MyPlugin {
	public function get_data() {
		return file_get_contents( DATA_DIR . 'data.txt' );
	}
}
```
Yay!</p>
```php
class MyPlugin {
	public $data_dir = '/data/dir';

	public function get_data() {
		return file_get_contents( $this->data_dir. 'data.txt' );
	}
}
```
					
---

				
					
Precept 4:</p>
Use filters to pass data indirectly.
					
---

				
					
Better to pass data directly, but...
					
---

				
					
When passing data between functions indirectly, use a filter. Filters can be mocked.
					
---

				
					
```php
function read_data( $post ) {
	add_filter( 'my_data', function() use ( &$post ) {
		return $post->post_content;
	} );
}

function return_data() {
	return apply_filters( 'my_data', $post->post_content );
}
```
You can write tests to be sure the filter is added and applied.
```php
add_filter( 'my_data', function() {
	return 'foobar';
} );
$this->assertEquals( 'foobar', return_data() );
```
					
---

				
					
Precept 5:</p>
Use verbs in all function names.
					
---

				
					
Isn't that just style?
					
---

				
					
A function name should tell you what inputs and outputs are expected.


Some great verbs are `get`, `is`, `update`, `create`, `remove`.

					
---

				
					
Example: a shortcode handler

`my_shortcode` doesn't really say what it does.

`process_my_shortcode` is better, but still ambiguous about what the function returns.

`get_markup_from_shortcode` tells us what the input and output will be.
					
---

				
					
If a function does too many things to name, consider splitting it.
					
---

				
					
Precept 6:</p>
Keep functions below ten lines and indentation below four levels.
					
---

				
					
Shorter functions are easier to understand.
					
---

				
					
Use `array_map`, `array_filter`, `array_reduce`, etc.</p><p> instead of `while`, `for`, and `foreach`.</p>
					
---

				
					
Nope.</p>
```php
function get_recent_post_titles( $posts ) {
	$matching = [];
	foreach( $posts as $post ) {
		if ( time() - strtotime( $post->post_date_gmt ) post_title;
		}
	}
	return $matching;
}
```
Yay!</p>
```php
function get_recent_post_titles( $posts ) {
	$posts = array_filter( $posts, 'is_post_recent' );
	return array_map( 'get_post_title', $posts );
}

function is_post_recent( $post ) {
	return ( time() - strtotime( $post->post_date_gmt ) post_title;
}
```
					
---

				
					
Precept 7:</p>
Consider all possible inputs and outputs.
					
---

				
					
Check for errors from external functions.
					
---

				
					
When testing (or in real life!), your data might be missing or incomplete.
					
---

				
					
Nope.</p>
```php
function get_post_title( $post_id ) {
	$post = get_post( $post_id );
	return $post->post_title;
}
```
Yay!</p>
```php
function get_post_title( $post_id ) {
	$post = get_post( $post_id );
	if ( $post ) {
		return $post->post_title;
	}
	return '';
}
```
					
---

				
					
Precept 8:</p>
Whenever possible, write tests first.
					
---

				
					
Tested code gives us confidence.

It will work the way we expect.


(We have defined our expectations.)


It can be refactored safely.

					
---

				
					
Precept 9:</p>
Don't test the code from other libraries, but test the inputs and outputs.
					
---

				
					
We can assume that WordPress works. (Otherwise we have bigger problems!)


But we should make sure that we are using WordPress the way we expect.

					
---

				
					
We can test the inputs to external functions with *spies*.
					
---

				
					
### What is a Spy?
An object that behaves like a function and which we can query to learn how it was called.
					
---

				
					
```php
$spy = \Spies\get_spy_for( 'add_shortcode' );

add_shortcode(
	'otherpages',
	'get_markup_from_shortcode'
);

$this->assertTrue(
	$spy->was_called_with(
		'otherpages',
		\Spies\any()
	);
);
```
					
---

				
					
Precept 10:</p>
Write only one assertion per test.
					
---

				
					
A unit is one thing. A unit test should test that one thing.<br/> One function's *one input* and *one output*.
					
---

				
					
Can a function have more than one input or output? Write more than one test.
					
---

				
					
> "Unit tests isolate failures. Even if a product contains millions of lines of code, if a unit test fails, you only need to search that small unit under test to find the bug."


https://googletesting.blogspot.co.uk/2015/04/just-say-no-to-more-end-to-end-tests.html

					
---

				
					
Code is hard. You can't predict all possible situations.

When you find a bug, write a test that causes it, and then fix the bug so the test passes.

					
---

				
					
### What is a unit test? (pt. 3)


Confidence, safety, readability.

