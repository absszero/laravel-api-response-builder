# API Response Builder for Laravel 5 #

Response Builder is Laravel5's helper designed to simplify building
nice, normalized and easy to consume REST API responses.


[![Latest Stable Version](https://poser.pugx.org/marcin-orlowski/laravel-api-response-builder/v/stable)](https://packagist.org/packages/marcin-orlowski/laravel-api-response-builder)
[![Latest Unstable Version](https://poser.pugx.org/marcin-orlowski/laravel-api-response-builder/v/unstable)](https://packagist.org/packages/marcin-orlowski/laravel-api-response-builder)
[![License](https://poser.pugx.org/marcin-orlowski/laravel-api-response-builder/license)](https://packagist.org/packages/marcin-orlowski/laravel-api-response-builder)

## Table of contents ##

 * [Response structure](#response-structure)
 * [Return Codes and Code Ranges](#return-codes)
 * [Exposed Methods](#exposed-methods)
 * [Usage examples](#usage-examples)
 * [Installation and Configuration](#installation-and-configuration)
 * [Handling Exceptions API way](#handling-exceptions-api-way)
 * [Manipulate Response Object](#manipulate-response-object)
 * [Overriding built-in messages](#overriding-built-in-messages)
 * [License](#license)
 * [Notes](#notes)


## Response structure ##

For simplicity of consuming, produced JSON response is **always** the same at its core and contains
the following data:

  * `success` (boolean) determines if API method succeeded or not
  * `code` (int) being your own return code
  * `locale` (string) locale used for error message (obtained automatically via \App::getLocale())
  * `message` (string) human readable description of `code`
  * `data` (object|null) your returned payload or `null` if there's no data to return.

If you need to return other/different fields in response, see [Manipulating Response Object](#manipulating-response-object).

## Return Codes ##

Return codes must be positive integer. Code `0` (zero) always means success, all
other codes are treated as error codes.


#### Code Ranges ####

In one of our projects we had multiple APIs chained together (so one API called another). In case of
method failure we wanted to be able to do the "cascade" and use return code provided by API that failed.
For example our API consumer call method of publicly exposed API "A". That API uses internal API "B"
method, but under the hood "B" also delegates some work and talks to API "C". In case of failure of
method in "C", API consumer would see its' return code. This simplifies the code, and helps keep features
separated but to make this work you must ensure no API return code overlaps, otherwise you cannot easily
tell which one in your chain failed. For that reason Response Builder supports code ranges, allowing you
to configure `min_code` and `max_code` you want to be allowed, and no code outside this range would be
allowed by Response Builder.

If you do not need code ranges for your API, just set `max_code` in configuration file to some very high value.

**IMPORTANT:** codes from `0` to `63` (inclusive) are reserved by Response Builder and cannot be assigned to your
codes.


## Exposed Methods ##

All ResponseBuilder methods are **static**, and for simplicity of use, it's recommended to
add the following `use` to make using Response Builder easier:

    use MarcinOrlowski\ResponseBuilder\ResponseBuilder;


Methods' arguments:

 * `$data` (mixed|null) data you want to be returned in response's `data` node,
 * `$http_code` (int) valid HTTP return code (see `HttpResponse` class for useful constants),
 * `$lang_args` (array) array of arguments passed to `Lang::get()` while building `message`,
 * `$error_code` (int) error code you want to be returned in `code`,
 * `$message` (string) custom message to be returned as part of error response.

Most arguments of `success()` and `error()` methods are optional, with exception for `$error_code`
for the latter. Helper methods arguments are partially optional - see signatures below for details.

**NOTE:** Since v2.1 you `$data` must no longer be `array`, but can literaly of any type, (i.e. `string`),
however to ensure returned JSON matches specification, data type conversion will be enforced
internally. There's no smart logic for doing that (with the exception for some Laravel types like
Model or Collection), so the result may not necessary be of your desire and you will end up with object
keys being "0" or "scalar".

**IMPORTANT:** If you want to use own `$http_code`, ensure it is right for the purpose.
Response Builder will throw `\InvalidArgumentException` if you use `$http_code` outside
of 200-299 range with `success()` and related methods and it will also do the same
for `error()` and related methods if `$http_code` will be lower than 400.

Redirection codes 3xx cannot be used with Response Builder.

See [W3 specs page](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html) for more details on HTTP codes.

#### Reporting Sucess ####

    success($data = null, $http_code = HttpResponse::HTTP_OK, array $lang_args = []);
    successWithHttpCode($http_code);

Usage restrictions:

* `$http_code` must be in range from 200 to 299

#### Reporting Error ####

    error($error_code, $lang_args = [], $data = null, $http_code = HttpResponse::HTTP_BAD_REQUEST);
    errorWithData($error_code, $data, array $lang_args = []);
    errorWithDataAndHttpCode($error_code, $data, $http_code, array $lang_args = []);
    errorWithHttpCode($error_code, $http_code, $lang_args = []);
    errorWithMessage($error_code, $error_message, $http_code = HttpResponse::HTTP_BAD_REQUEST);

Usage restrictions:

* `$error_code` must not be 0
* `$http_code` must not be lower than 400


## Usage examples ##

#### Success ####

To report success from your API, just conclude your Controller with:

    return ResponseBuilder::success();

which will produce:

     {
       "success": true,
       "code": 0,
       "locale": "en",
       "message": "OK",
       "data": null
     }

If you want to return some data back too, pass it to `success()`:

    $data = [ "foo" => "bar" ];
    return ResponseBuilder::success($data);

which would return:

    {
      ...
      "data": {
         "foo": "bar"
      }
    }
    
Since v2.1 you can return Eloquent model object directly:

    $flight = App\Flight::where('active', 1)->first();
    return ResponseBuilder::success($flight);

which would return model attributes (`toArray()` will automatically be called by Response Builder). The
imaginary output would then look like this:

    {
      "item": {
          "airline": "lot",
          "flight_number": "lo123",
          ...
       }
    }

You can also return whole Collection:

    $flights = App\Flight::where('active', 1)->get();
    return ResponseBuilder::success($flights);

which would return array of objects:

    {
      "items": [
            {
               "airline": "lot",
               "flight_number": "lo123",
               ...
            },{
               "airline": "american",
               "flight_number": "am456",
               ...
            }
         ]
       }
    }

    
`item` and `items` keys are configurable (see `app/config/response_builder.php` or if you already
published config file, look into your `vendor/marcin-orlowski/laravel-api=response-builder/config/`
folder for new items)

**NOTE:** currently there's no recursive processing, so if you want to return Eloquent 
model as part of own array structure you must explicitely call `toArray()` on such object
prior adding it to your array you want to pass to Response Builder:

    $data = [ 'flight' = App\Flight::where('active', 1)->first()->toArray() ];
    return ResponseBuilder::success($data);


**IMPORTANT:** `data` node is **always** returned as JSON Object. This is enforced by design, therefore trying to return
plain array:

    $returned_array = [1,2,3];
    return ResponseBuilder::success($returned_array);

will produce:

    {
      ...
      "data": {
         "0": 1,
         "1": 2,
         "2": 3
      }
    }

To avoid this you need to make the array part of object, which
in usually means wrapping it in another array:

    $returned_array = [1,2,3];
    $data = ['my_array' => $returned_array];
    return ResponseBuilder::success($data);

This would produce expected and much cleaner data structure:

    {
      ...
      "data": {
         "my_array": [1, 2, 3]
      }
    }


#### Errors ####

Returning errors is almost as simple as returning success, however you need to provide at least error
code to report back to caller. To keep your source readable and clear, it's strongly suggested to create separate 
class i.e. `app/ErrorCode.php` and put all codes you need to use in your code there:

    <?php

    namespace App;

    class ErrorCode {
        const SOMETHING_WENT_WRONG = 250;
    }

To report failure of your method just do:

    return ResponseBuilder::error(ErrorCode::SOMETHING_WENT_WRONG);

and you will produce:

    {
      "success": false,
      "code": 250,
      "locale": "en",
      "message": "Error #250",
      "data": null
    }

Response Builder tries to automatically obtain error message for each code. This is
configured in `config/response_builder.php` file, with use of `map` array. See
[Response Builder Configuration](#response-builder-configuration) for more details.

If there's no dedicated message configured, `message` is populated with built-in generic
fallback message "Error #xxx", as shown above.

As Response Builder uses Laravel's Lang package, you can use the same features with
your messages as you use across the whole app, incl. using placeholders:

    return ResponseBuilder::error(ErrorCode::SOMETHING_WENT_WRONG, ['login' => $login]);

However this is not recommended, you can override message mapping by providing replacement
error message with use of `errorWithMessage()`, which expects string message as argument.
In this case however, if placeholders are used, you need to resolve them yourself:

    $msg = Lang::get('message.something_wrong', ['login' => $login]);
    return ResponseBuilder::errorWithMessage(ErrorCode::SOMETHING_WENT_WRONG, $msg);


## Installation and Configuration ##

To install Response Builder all you need to do is to open your shell and do:

    composer require marcin-orlowski/laravel-api-response-builder

If you want to change defaults then you should publish configuration file to 
your `config/` folder:

    php artisan vendor:publish

and tweak according to your needs. If you are fine with defaults, this step
can safely be skipped.


#### Laravel setup ####

Edit `app/config.php` and add the following line to your `providers` array:

    MarcinOrlowski\ResponseBuilder\ResponseBuilderServiceProvider::class,


#### Response Builder Configuration ####

Response Builder configuration can be found in `config/response_builder.php` file. Supported
configuration keys (all must be present):

 * `min_code` (int) lowest allowed code for assigned code range (inclusive)
 * `max_code` (int) highest allowed code for assigned code range (inclusive)
 * `map` (array) maps error codes to localization string keys.

Code to message mapping example:

    'map' => [
        ErrorCode::SOMETHING => 'api.something',
    ],

If given error code is not present in `map`, Response Builder will provide fallback message automatically 
(default message is like "Error #xxx"). This means it's perfectly fine to have whole `map` array empty in
your config, however you must have `map` key present:

    'map' => [],

Also, read [Overriding built-in messages](#overriding-built-in-messages) to see how to override built-in
messages.


## Messages and Localization ##

Response Builder is designed with localization in mind so default approach is you just set it up
once and most things should happen automatically, which also includes creating human readable error messages.
As described in `Configuration` section, once you got `map` configured, you most likely will not
be in need to manually refer error messages - Response Builder will do that for you and you optionally
just need to pass array with placeholders' substitution (hence the order of arguments for `errorXXX()`
methods). Response Builder utilised standard Laravel's `Lang` class to deal with messages, so all features
localization are supported.


## Handling Exceptions API way ##

Properly designed API shall never hit consumer with HTML nor anything like that. While in regular use this
is quite easy to achieve, unexpected problems like uncaught exception or even enabled maintenance mode
can confuse many APIs world wide. Do not be one of them and take care of that too. With Laravel this
can be achieved with custom Exception Handler and Response Builder comes with ready-to-use Handler as
well. See [EXCEPTION_HANDLER.md](EXCEPTION_HANDLER.md) for easy setup information.


## Manipulating Response Object ##

If you need to return more fields in response object you can simply extend `ResponseBuilder` class
and override `buildResponse()` method:

    protected static function buildResponse($code, $message, $data = null);

For example, to remove `locale` field but add server time and timezone to returned response create
`MyResponseBuilder.php` file in `app/` folder with content:

    <?php

    namespace App;

    class MyResponseBuilder extends MarcinOrlowski\ResponseBuilder\ResponseBuilder
    {
        protected static function buildResponse($code, $message, $data = null)
        {
            // tell ResponseBuilder to do the dirty job
            $array = parent::buildResponse($code, $message, $data);

            // and tweak the response structure as you wish
            $date = new DateTime();
            $array['timestamp'] = $date->getTimestamp();
            $array['timezone'] = $date->getTimezone();
            unset($array['locale']);

            return $date;
        }
    }

and from now on use your class instead:

    MyResponseBuilder::success();

which would return:

     {
       "success": true,
       "code": 0,
       "message": "OK",
       "timestamp": 1272509157,
       "timezone": "UTC",
       "data": null
     }


## Overriding built-in messages ##

At the moment Response Builder provides few built-in messages (see [src/ErrorCode.php](src/ErrorCode.php)):
one is used for success code `0` and anothers provide fallback message for codes without custom mapping. If for 
any reason you want to override them, simply map these codes in your `map` config using codes from package
resrved range:

     MarcinOrlowski\ResponseBuilder\ErrorCode::OK => 'my_messages.ok',

and from now on, each `success()` will be returning your message instead of built-in one.

To override default error message used when given error code has no entry in `map`, add the following:

     MarcinOrlowski\ResponseBuilder\ErrorCode::NO_ERROR_MESSAGE => 'my_messages.default_error_message',

You can use `:error_code` placeholder in the message and it will be substituted actual error code value.


## License ##

* Written and copyrighted by Marcin Orlowski
* Response Builder is open-sourced software licensed under the [MIT license](http://opensource.org/licenses/MIT)


## Notes ##

* Response Builder is **not** compatible with Lumen framework, mainly due to lack of Lang class. If you would like to help making Response Builder usable with Lumen, speak up or (betteR) send pull request!
* Tests will be released shortly. They do already exist, however Response Builder was extracted from existing project and making tests work again require some work to remove dependencies.
