A PHP REST server for providing a very light-weight REST API. Very easy to set up and get going. Independent from other libraries and frameworks. Supports HTTP authentication.

Initial Documentation from http://jacwright.com/250/simple-rest-server-in-php-supports-json-amf/

## Simple REST server in PHP

After building a couple of RESTful services using the [Zend Framework](http://framework.zend.com/), I decided to create a dead simple REST server that allowed me to skip all the features I didn’t need as well as a **tons of classes** that came with Zend Framework MVC. There are still useful features to add (XML support for example), but overall I’m quite happy with what I’ve come up with.

My solution, `RestServer`, is a JSON REST server, so far. It should be trivial to add support for XML or other formats, but there would have to be assumptions on what your object would look like in XML (XML-RPC style, your own custom XML format, etc). First we’ll look at the classes that you write to handle the requests, then we’ll look at how to tie it together in your index.php file.

### REST Controllers

The `RestServer` class assumes you are using URL rewriting and looks at the URL from the
request to map to the necessary actions. The map that gets a request from URL to class
method is all in the doc-comments of the classes. Here is an example of a class that
would handle some user actions:

```php
class TestController
{
    /**
     * Returns a JSON string object to the browser when hitting the root of the domain
     *
     * @url GET /
     */
    public function test()
    {
        return "Hello World";
    }

    /**
     * Logs in a user with the given username and password POSTed. Though true
     * REST doesn't believe in sessions, it is often desirable for an AJAX server.
     *
     * @url POST /login
     */
    public function login()
    {
        $username = $_POST['username'];
        $password = $_POST['password'];
        // validate input and log the user in
    }

    /**
     * Gets the user by id or current user
     *
     * @url GET /users/:id
     * @url GET /users/current
     */
    public function getUser($id = null)
    {
        if ($id) {
            $user = User::load($id); // possible user loading method
        } else {
            $user = $_SESSION['user'];
        }

        return $user; // serializes object into JSON
    }

    /**
     * Saves a user to the database
     *
     * @url POST /users
     * @url PUT /users/:id
     */
    public function saveUser($id = null, $data)
    {
        // ... validate $data properties such as $data->username, $data->firstName, etc.
        $data->id = $id;
        $user = User::saveUser($data); // saving the user to the database
        return $user; // returning the updated or newly created user object
    }
}
```

Let’s walk through the above `TestController` class to talk about the features demonstrated. First we’ll look at the `test` method. You’ll notice there is a new kind of doc-comment tag in the docblock. `@url` maps a URL to the method below it and is in the form:

`@url <REQUEST_METHOD> <URL>`

In this particular example, when someone does a GET on http://www.example.com/ (assuming example.com is where our service is located) it will print out:

`"Hello World"`

which is a valid representation of a string in JSON.

Moving on to the next method, `login`, we see the `@url` maps any POSTs to http://www.example.com/login to the `login` method. Getting data from a regular web-type POST is the same as any PHP application, allowing you to use your own validation or other framework in conjunction with this REST server. Sessions can also be kept if desired. Though keeping sessions isn’t true REST style, often all we want a REST server for is to serve up data to our ajax application, and it can be easier to just use sessions than something more RESTful.

Next we have our `getUser` method (you’ll notice that it doesn’t really matter what I name my methods because our `@url` directives define what URLs map to the method). You can see a couple of things here. First, we have multiple `@url` mappings for this method. And second, there is an odd `/:id` in that first URL mapping. RestServer treats any `:keyword` placeholders as wildcards in the URL and will take that section of the URL and pass it into the parameter with the same name in the method. In this example, when hitting http://www.example.com/users/1234, `$id` will equal 1234. When hitting http://www.example.com/users/current, `$id` will equal null. It doesn’t matter what order your parameters are in, so long as they have the same name as the placeholder (`:id` and `$id`, `:username` and `$username`). You’ll also want to be sure to make your parameters optional (`$id = null`) when you have several URL mappings that don’t all require a parameter. Otherwise you’ll have an error thrown telling you that you didn’t pass in a required parameter.

One last thing to note in `getUser` is that this method simply returns a `User` object. This gets serialized into JSON (or potentially XML) and printed out for consumption by the application.

Finally we get to `saveUser`. You see here we have multiple URL mappings again. This time they also have different HTTP methods (POST and PUT) for creating and updating a user. The new thing here is the `$data` variable. This is a special keyword parameter that will contain the value of whatever was POSTed or PUT to the server. This is different than your regular web POST in that it doesn’t need to only be name-value pairs, but can be as robust as JSON, sending complex objects. For example, the body of a regular web POST, let’s say the login request, might look like this:

`username=bob&password=supersecretpassw0rd`

but POSTing a new user object for our saveUser method could look like this:

```json
{ "username": "bob", "password": "supersecretpassword", "firstName": "Bob", "lastName": "Smith" }
```

So you’re able to allow POSTing JSON in addition to regular web style POSTs.

I call these classes that handle the requests `Controllers`. And they can be completely self-contained with their URL mappings, database configs, etc. so that you could drop them into other RestServer services without any hassle.

### REST index.php

In order to get the whole server kicked off, you’ll want to create an index.php file, have your URL rewriting direct requests to it (another topic which you can learn about elsewhere), and create the RestServer and add controller classes to it for handling. RestServer will cache the URL mappings between requests using APC or a file to speed up requests. You won’t have to load every controller file on every request if you use autoload and this cache, only the one needed for the request. The cache only runs in production mode. Here is an example index.php file:

```php
spl_autoload_register(); // don't load our classes unless we use them

$mode = 'debug'; // 'debug' or 'production'
$server = new RestServer($mode);
// $server->refreshCache(); // uncomment momentarily to clear the cache if classes change in production mode

$server->addClass('TestController');
$server->addClass('ProductsController', '/products'); // adds this as a base to all the URLs in this class

$server->handle();
```

That’s it. You can add as many classes as you like. If there are conflicts, classes added later will overwrite duplicate URL mappings that were added earlier. And the second parameter in addClass can be a base URL which will be prepended to URL mappings in the given class, allowing you to be more modular.

You can [view the RestServer class](https://github.com/jacwright/RestServer/blob/master/RestServer.php), copy it and use it for your own purposes. It is under the MIT license. Features to be added include XML support and HTTP Authentication support. If you make this class better please share your updates with everyone by leaving a comment. I will try and keep this class updated with new features as they are shared. I hope you enjoy!

I changed the title of this post to remove the AMF portion but was asked to cover it, so I will quickly talk about the AMF support. RestServer supports the AMF format in addition to the JSON format. This is a binary format used by Adobe Flash in their remoting services, but because their remoting services are not RESTful, you can’t use classes such as RemoteObject with REST.

In order to use the AMF format with this service, you’ll need to have the Zend Framework in your classpath so that the classes to serialize and deserialize AMF are present (e.g. Zend/Amf/Parse/Amf3/Serializer.php). Then you’re ready so server up AMF. The way you consume AMF in Flash is using URLLoader or URLStream to load the data and ByteArray to convert it into an object. To tell the server you want AMF rather than JSON your URLRequest object will need to add an Accept header of “application/x-amf”. Below I will show you how this could be done.

```php
public function getUser():void
{
    var loader:URLLoader = new URLLoader();
    loader.dataFormat = URLLoaderDataFormat.BINARY;
    var request:URLRequest = new URLRequest("http://www.example.com/users/current");
    request.requestHeaders.push(new URLRequestHeader("Accept", "application/x-amf"));
    loader.addEventListener(Event.COMPLETE, onComplete);
    loader.load(request);
}

private function onComplete(event:Event):void
{
    var loader:URLLoader = event.target as URLLoader;
    var byteArray:ByteArray = loader.data as ByteArray;
    var user:Object = byteArray.readObject();
    // do something with the user
}
```

You can even make your PHP objects cast into their actionscript equivalents using the $_explicitType property or getASClassName method in PHP as defined in the [documentation](http://framework.zend.com/manual/en/zend.amf.server.html#zend.amf.server.typedobjects) and by using registerAlias in Flash.

Good luck and let me know if you end up using it!

**Update:** I am including an example .htaccess file for anyone who might need it. It will only rewrite requests to files that don’t exist, so you can have images, css, or other PHP files in your webroot and they will still work. Anything that would give a 404 will redirect to your index.php file.

```
DirectoryIndex index.php
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule ^$ index.php [QSA,L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule ^(.*)$ index.php [QSA,L]
</IfModule>
```
