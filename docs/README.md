# Getting started

__Xenus__ is a simple yet powerfull ODM for MongoDB. It uses the latest technologies such as PHP 7 and the new MongoDB driver.

## Installation

Xenus requires PHP version 7.0 or higher and the MongoDB driver version 1.2 or higher.

Make sure to install first the MongoDB driver : http://php.net/manual/en/mongodb.installation.php

Once installed, run the following from your project root :

```bash
composer require abellion/xenus
```

## First steps

> This guide assumes you have an existing knowledge of the MongoDB library and driver.

- Driver documentation : http://php.net/manual/en/set.mongodb.php
- Library documentation : https://docs.mongodb.com/php-library/v1.2/

Before going further, make sure you have a running MongoDB server and a established connection with it :

```php
use MongoDB;

$client = new MongoDB\Client()

$database = $client->myDatabase;

```

> To know more about the `Client`, `Database` and `Collection` classes, please refer to the MongoDB library : https://docs.mongodb.com/php-library/v1.2/reference/class/MongoDBClient/

# Guide

## Collections

### Defining collections

Having a collection of users is very common, so let's take that as an example :

```php
use Xenus\Collection;

class Users extends Collection
{
    //This is the name of your collection stored in the database
    protected $name = 'users';
}
```

A Xenus `Collection` needs two properties to be constructed :

- An instance of the `MongoDB\Database` class,
- The name of the collection as a protected property.

```php
use MongoDB;

$client = new MongoDB\Client()
$database = $client->myDatabase;

$users = new Users($database);
```

> If you use a framework with a dependency injection container, you can configure it to automatically inject an instance of the `MongoDB\Database` in every collections.

### Retrieving documents

Collections inherit from the `MongoDB\Collection` class so you can use any of the methods provided by the MongoDB library : https://docs.mongodb.com/php-library/v1.2/reference/class/MongoDBCollection/

| Method | Description |
|--------|-------------|
| `find()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-find/ |
| `findOne()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-findOne/ |

```php
$adultUsers = $users->find([
    'age' => ['$gte' => 21]
]);
```

Instead of making this query everywhere you need these users, you may better make a `findAdults()` method inside the users collection :

```php
use Xenus\Collection;

class Users extends Collection
{
    protected $name = 'users';

    public function findAdults(array $options = [])
    {
        return $this->find([
            'age' => ['$gte' => 21]
        ], $options);
    }
}
```

```php
$adultUsers = $users->findAdults();
```

Because we added the `$options` parameter, we can simply sort the results :

```php
$adultUsers = $users->findAdults([
    'sort' => ['age' => 1]
]);
```

You'll see you very often retrieving models by ID typing `['_id' => $id]`. To avoid that, Xenus allows you to give directly a `MongoDB\BSON\ObjectID` instance that's automatically translated to the array form :

```php
// Typing
$users->findOne($id);
// Is the same as :
$users->findOne(['_id' => $id]);
```

### Inserting & Updating documents

The MongoDB library offers you some great methods to work with update and delete operations :

| Method | Description |
|--------|-------------|
| `insertOne()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-insertOne/ |
| `insertMany()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-insertMany/ |
| `updateOne()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-updateOne/ |
| `updateMany()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-updateMany/ |
| `deleteOne()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-deleteOne/ |
| `deleteMany()` | https://docs.mongodb.com/php-library/v1.2/reference/method/MongoDBCollection-deleteMany/ |

On top of these methods, Xenus provides some more natural `update()`, `delete()` and `insert()` methods to work with single documents. These methods take directly a document as parameter :

```php
$john = ['_id' => '...', 'name' => 'John'];

//Create
$users->insert($john);

//Update
$users->update($john);

//Delete
$users->delete($john);
```

Sometimes you don't want or need to retrieve a document before updating or deleting it, so you will use one of the `updateOne()` and `deleteOne()` methods. Because it's so common to use them with a single `_id` filter, Xenus offers you the possibility to give directly a `MongoDB\BSON\ObjectID` :

```php
//Update directly by ID
$users->updateOne($userId, [...]);

//Delete directly by ID
$users->deleteOne($userId)
```

## Documents

Instead of receiving your data in the form of an array or a pristine object, Xenus allows you to treat your documents as pre-defined objects, containing a lot of helpful methods.

### Creating a document

A document is no more than a class containing your document's properties stored in your MongoDB database.
A user document may contain a `name`, an `email`, a `username` or whatever is fitting your business.

```php
use Xenus\Document;

class User extends Document
{
    protected $withId = true;
}
```

> The `withId` property, when set to `true`, will create an `_id` field containing a fresh `MongoDB\BSON\ObjectID` instance.

To create a new user, simply instantiate the `User` object, optionnaly with a set of properties :

```php
//It creates an empty user, and then sets a name
$john = new User();
$john->name = 'John';

//It creates directly the user with a name
$john = new User(['name' => 'John']);
```

When retrieving models, you'll receive them as a `Xenus\Document` by default. To tell Xenus to use a custom document, you need to set a property called `document` in the `Collection` class :

```php
use Xenus\Collection;

class Users extends Collection
{
    protected $name = 'users';
    protected $document = User::class;
}
```

From now, whenever you'll fetch some users, they will come in the form of the `User` class.

### Accessing properties

The `Xenus\Document` class implements the `ArrayAccess` interface and uses the magic methods `__get` and `__set`. You can access your document's properties using these ways :

```php
//Get the user name
$user->name;
$user['name'];

//Set the user name
$user->name = 'John';
$user['name'] = 'John';
```

On top of that, a document offers you three methods `get()`, `set()` and `has()` to work in a more OOP way :

```php
//Return a boolean
$user->has('name');

//Return the document's name value
$user->get('name');

//Set the document's name value and return the instance
$user->set('name', $name);
```

If you want to validate the data before setting it, or tweak the data before getting it, you can declare a set of accessors (a getter and a setter) in your models.

Doing this way has several advantages :

- At first glance you can see what's inside a model
- You can take advantage of PHP type-hint to validate the input
- You can tweak the data (like applying a `trim()`)

```php
use Xenus\Document;

class User extends Document
{
    public function getName()
    {
        return $this->get('name');
    }

    public function setName(string $name)
    {
        return $this->set('name', trim($name));
    }
}
```

When mutating a document, Xenus will try to find the corresponding getter and setter for you so you don't have to use the quite verbose `setXXX()` and `getXXX` methods.

- A getter is a method whose name starts with `get` and its followed by the property name.
- A setter is a method whose name starts with `set` and its followed by the property name.

Now, when typing `$user->name` or `$user->name = 'Antoine'`, the `getName()` and `setName()` methods will be used.

!> This "auto discovery" mechanism is not used if you use the array form (ie. `$model['property']`) to keep a way of muttating the documents without using the accessors.

Of course, as well as these accessors, you can add some other methods serving your business logic.

```php
use Xenus\Document;

class User extends Document
{
    public function isAdult()
    {
        return $this->get('age') >= 21;
    }
}
```

### Embedding documents

One super feature of MongoDB is the ability to nest data within documents. For example, a user could have some contact informations stored as the following document :

```php
use Xenus\Document;

class Contact extends Document
{
    public function getPhoneNumber()
    {
        return $this->get('phone_number');
    }

    public function setPhoneNumber($phoneNumber)
    {
        return $this->set('phone_number', $phoneNumber);
    }
}
```

To easily embed this `Contact` document into your user model, you'll find the `Embed` utility in the `Xenus\Support` namespace. It contains a single `document()` method :

- `Embed::document(Contact::class)->on($contact)` : It makes the `$contact` be an instance of the `Contact` document.

- `Embed::document(Contact::class)->in($contact)` : It makes every of the `$contact` values be an instance of the `Contact` document.

In short : the `on()` method instantiate the given class with the given value, the `in()` method does the same but on every value of the given array.

A user might have one primary contact, and many other as a backup :

```php
use Xenus\Document;
use Xenus\Support\Embed;

class User extends Document
{
    public function getContacts()
    {
        return $this->get('contacts');
    }

    public function setContacts(array $contacts)
    {
        return $this->set('contacts', Embed::document(Contact::class)->in($contacts));
    }

    public function getPrimaryContact()
    {
        return $this->get('primary_contact');
    }

    public function setPrimaryContact($contact)
    {
        return $this->set('primary_contact', Embed::document(Contact::class)->on($contact));
    }
}
```

- Its `contacts` value will be an array of `Contact` documents.

- Its `primary_contact` value will be an instance of the `Contact` document.

Retrieving the contact informations is then as fluent as any other document's attribute :

```php
$phoneNumber = $user->primaryContact->phoneNumber;
```

Of course, you're not limited to embed documents. You could have an array of `ObjectID` and make the following to be sure it contains only `ObjectID` values :

```php
public function setFirendsId(array $friendsId)
{
    return $this->set('friends_id', Embed::document(ObjectID::class)->in($friendsId));
}
```

## Relationships

Embedding documents does not cover every use case. When you need to reference a document from another, you store its unique identifier - in one of the side of the relationship - and then retrieve the referenced document.

__Terminology :__

Given the situation "A User has a Profile" :

- The "User" is the `parent` of the relationship,
- The "Profile" is the `child` of the relationship.

- The "User" `has` a "Profile",
- The "Profile" `belongs to` a "User".

### One To One

One To One relationships are very basic relations. For example, a user might have one address. To define a One To One relationship we use the `hasOne` method on the `parent` side and the `belongsTo` method on the `child` side :

```php
use Xenus\Document;

class User extends Document
{
    public function getAddress()
    {
        return $this->hasOne(Addresses::class, 'user_id');
    }
}

class Address extends Document
{
    public function getUser()
    {
        return $this->belongsTo(Users::class, 'user_id');
    }
}
```

In this example :

- The `hasOne` method will look in the "Addresses" collection for an address with a `user_id` corresponding to the user unique identifier.

- The `belongsTo` method will look in the "Users" collection for a user with a unique identifier corresponding to its `user_id` value.

### One To Many

One To Many relationships are used when something is linked to many other things. For exemple, a user might have many addresses. To define a One To Many relationship, we use the `hasMany` method on the `parent` side and the `belongsTo` method on the `child` side :

```php
use Xenus\Document;

class User extends Document
{
    public function getAddresses()
    {
        return $this->hasMany(Addresses::class, 'user_id');
    }
}

class Address extends Document
{
    public function getUser()
    {
        return $this->belongsTo(Users::class, 'user_id');
    }
}
```

In this example :

- The `hasMany` method will look in the "Addresses" collection for all the addresses with a `user_id` corresponding to the user unique identifier.

- The `belongsTo` method will look in the "Users" collection for a user with a unique identifier corresponding to its `user_id` value.

### Many To Many

Taking the previous example, what if some other users had the same address ? A user would have many addresses, and each address would have many users. To define a Many To Many relationship, we use on both sides the `belongsToMany` method :

```php
use Xenus\Document;

class User extends Document
{
    public function getAddresses()
    {
        return $this->belongsToMany(Addresses::class, 'addresses_id');
    }
}

class Address extends Document
{
    public function getUsers()
    {
        return $this->belongsToMany(Users::class, 'users_id');
    }
}
```

In this example :

- The user's `belongsToMany` method will look for the addresses whose unique identifier are those stored in its `addresses_id` field.

- The address's `belongsToMany` method will look for the users whose unique identifier are those stored in its `users_id` field.

### Querying relations

Every of the relationship methods (`hasOne`, `belongsTo`, ...) return an object containing a `find()` method. Calling it will return a single document or a cursor depending on the relation type.

Given the previous examples, querying the user for its address can be done with the following :

```php
$address = $user->getAddress()->find();

// Also by taking advantage of getters :

$address = $user->address->find();
```

Additionally, you may give a filter, for example to retrieve the user's address only if it's located in Paris :

```php
$address = $user->address->find([
    'city' => 'Paris'
]);
```

!> A document's relations work only if the document comes from the `Collection` class (after a `find()` or `findOne()` call). Meaning that instantiating a document by hand will not allow to use relationships.

In the case of a `hasMany` and `belongsToMany` relationships, you also have access to the `findOne`, `count` and `distinct` methods.

These methods take the same parameters as the ones you know in the `Collection` class.

## Resources

When making an API, you expose your database resources to the outside world (the users, the blog posts, etc...).

Thus, you need to make sure none of the private attributes are going outside (a user password for example). You also may want to convert some attributes. For example, a `MongoDB\BSON\ObjectID` into a beautiful string called `id` instead of `_id`.

Let's create a pristine resource representing a user :

```php
use Xenus\Document;

class UserResource extends Document
{

}
```

### Selecting attributes

Imagine you want to keep from your user only a certain set of attributes, for example its `name` and `city` - but none of its `password`, `email` or anything sensitive.

Xenus documents have a `with()` method that selectivly keep the given attributes :

```php
$user = $user->with(['name', 'city']);
```

But instead of listing these attributes in every controllers you return a user, let's centralize into the `UserResource` :

```php
use Xenus\Document;

class UserResource extends Document
{
    public function __construct(User $user)
    {
        parent::__construct(
            $user->with(['name', 'city'])
        );
    }
}
```

In your controllers, you can now return an instance of the `UserResource` that do the work behind the scene :

```php
return new UserResource(
    $users->findOne([ ... ])
);
```

### Transforming attributes

Your models may contain a `createdAt` timestamp storing the model's creation date, in seconds. On the client, Javascript side, timestamps are treated in miliseconds. To make client side development easier, returning a milisecond timestamp would be great. By taking advantage of Xenus setters, making this tweak is very easy, you juste need to add a `setCreatedAt` setter inside the `UserResource` :

```php
use Xenus\Document;

class UserResource extends Document
{
    public function setCreatedAt($createdAt)
    {
        return $this->set('created_at', $createdAt * 1000);
    }
}
```

Now, when returning this `userResource` from your controllers, the `created_at` attribute will always be a milisecond timestamp without any work !

It's also common to tweak a model's ID, to rename the `_id` attribute into `id` and to make its `MongoDB\BSON\ObjectID` value a string :

```php
use Xenus\Document;

class UserResource extends Document
{
    public function setId($id)
    {
        return $this->set('id', (string) $id);
    }
}
```

### Using a resource

When returning a single resource, it's ok to make the following :

```php
return new CommentResource($comment);
```

On the opposite, it may become cumbersome when dealing with collections, a list of comments for example. You would need to loop over the comments to make every of them a `CommentResource`.

Instead, you may better use the `Xenus\Support\Transform` utility :

```php
use Xenus\Support\Transform;

return Transform::collection($comments)->to(CommentResource::class);
```

It accepts arrays as well as cursors, so you can transform directly the result of a `find()` query :

```php
use Xenus\Support\Transform;

return Transform::collection(
    $comments->find()
)->to(CommentResource::class);
```

To be consistant, you can also transform single documents using the `document()` method :

```php
use Xenus\Support\Transform;

return Transform::document($comment)->to(CommentResource::class);
```

