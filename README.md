# Doctrine ORM w/ Laravel 
> extracted from this [blog](https://isaacearl.com/blog/laravel-doctrine-setup)

Relevant links:
- http://www.laraveldoctrine.org/docs/1.3/orm/installation
- http://www.laraveldoctrine.org/docs/1.3/migrations/installation
- http://www.laraveldoctrine.org/docs/1.3/extensions/installation

**Install the laravel-doctrine/extensions package and the gedmo one. We don't need the Beberlei extension.**

> Note: as you see below we are using v1.4 as the laravel used for this project is v5.8

```bash
composer require "laravel-doctrine/orm:1.4.*"
composer require "laravel-doctrine/migrations"
composer require "laravel-doctrine/extensions:1.0.*"
composer require "gedmo/doctrine-extensions=^2.4"
```
After updating composer, add the ServiceProvider() entries to the providers array in `config/app.php`

```php
LaravelDoctrine\ORM\DoctrineServiceProvider::class,
LaravelDoctrine\Migrations\MigrationsServiceProvider::class,
# needed for annotation driver
LaravelDoctrine\Extensions\GedmoExtensionsServiceProvider::class,

```
Publish the config:

```php
php artisan vendor:publish --tag="config"
```

## Setup
Create an Entities folder in /app and add a User:
```php
<?php

namespace App\Entities;

use Doctrine\ORM\Mapping as ORM;
use Illuminate\Auth\Passwords\CanResetPassword;
use Illuminate\Contracts\Auth\Authenticatable as AuthenticatableContract;
use Illuminate\Contracts\Auth\CanResetPassword as CanResetPasswordContract;
use LaravelDoctrine\ORM\Auth\Authenticatable;
use LaravelDoctrine\Extensions\Timestamps\Timestamps;
use LaravelDoctrine\ORM\Notifications\Notifiable;

/**
 * @ORM\Entity
 * @ORM\Table(name="users")
 */
class User implements AuthenticatableContract, CanResetPasswordContract
{

    use Authenticatable, CanResetPassword, Timestamps, Notifiable;

    /**
     * @ORM\Id
     * @ORM\GeneratedValue(strategy="IDENTITY")
     * @ORM\Column(type="integer")
     */
    protected $id;

    /**
     * @var string
     * @ORM\Column(type="string")
     */
    protected $email;

    /**
     * @var string
     * @ORM\Column(type="string",nullable=false)
     */
    protected $name;
/**
     * @return mixed
     */
    public function getId()
    {
        return $this->id;
    }
    /**
     * @return string
     */
    public function getEmail()
    {
        return $this->email;
    }
    /**
     * @param string $email
     */
    public function setEmail($email)
    {
        $this->email = $email;
    }
    /**
     * @return string
     */
    public function getName()
    {
        return $this->name;
    }
    /**
     * @param string $name
     */
    public function setName($name)
    {
        $this->name = $name;
    }
    /**
     * @return int
     */
    public function getKey()
    {
        return $this->getId();
    }
}
```

Fix the namespace and paths in `doctrine.php` to point at where you put your Entities...
        
> If you don't have this file, you need to publish the config.
```php
# doctrine.php
    'namespaces' => [
            'App\Entities'
    ],
    'paths' => [
            base_path('app/Entities')
    ],
```

Uncomment the Timestampable extension as well as any others you might want to use.
```php
    'extensions'                 => [
    //LaravelDoctrine\ORM\Extensions\TablePrefix\TablePrefixExtension::class,
    LaravelDoctrine\Extensions\Timestamps\TimestampableExtension::class,
    LaravelDoctrine\Extensions\SoftDeletes\SoftDeleteableExtension::class,
    //LaravelDoctrine\Extensions\Sluggable\SluggableExtension::class,
    //LaravelDoctrine\Extensions\Sortable\SortableExtension::class,
    //LaravelDoctrine\Extensions\Tree\TreeExtension::class,
    //LaravelDoctrine\Extensions\Loggable\LoggableExtension::class,
    //LaravelDoctrine\Extensions\Blameable\BlameableExtension::class,
    //LaravelDoctrine\Extensions\IpTraceable\IpTraceableExtension::class,
    //LaravelDoctrine\Extensions\Translatable\TranslatableExtension::class
    ],
```
Now go to your `auth.php` file

change the providers section to use your entity and the doctrine driver.
```php
# auth.php
    'providers' => [
        'users' => [
            'driver' => 'doctrine',
            'model' => App\Entities\User::class,
        ],
    ],
```
Finally, go ahead and replace the PasswordResetServiceProvider provided by Laravel, with the one provided by laravel-doctrine in the `app.php` file as well.
```php
//        Illuminate\Auth\Passwords\PasswordResetServiceProvider::class,
        LaravelDoctrine\ORM\Auth\Passwords\PasswordResetServiceProvider::class,
```

Now we are ready to scaffold out the authentication stuff. Run the make auth command:
```bash
php artisan make:auth
```

Go ahead and delete the 2 migrations that get created with this command in the migrations/ folder. After you delete them, then we'll generate our own migration for our user entity.

```bash
php artisan doctrine:migrations:diff
```
then migrate it

```bash
php artisan doctrine:migrations:migrate
```
> Note: you'll notice we don't create a migration or entity for the password_resets table. We don't actually need one, Doctrine will magically create one when it tries to use it for the first time.

Now you just have to make a few more modifications to the generated controllers to get everything to work. Go to the `RegisterController.php` and modify the validator function. You must use the unique rule slightly different with laravel doctrine. [here](http://www.laraveldoctrine.org/docs/1.3/orm/validation)

It should look like this (change the reference to the users table to your User entity)

```php
# RegisterController.php

use App\Entities\User;
use EntityManager;
use Illuminate\Support\Facades\Validator;
...

    protected function validator(array $data)
    {
        return Validator::make($data, [
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:App\Entities\User',
            'password' => 'required|string|min:6|confirmed',
        ]);
    }
```
Change the create() function so it uses the new entity to create the user.

```php
    protected function create(array $data)
    {

        $user = new User();
        $user->setEmail($data['email']);
        $user->setName($data['name']);
        $user->setPassword(bcrypt($data['password']));

        EntityManager::persist($user);
        EntityManager::flush();

        return $user;
    }
```
Then in the `ResetPasswordController.php` override the resetPassword function by adding this function to the controller:

```php
# ResetPasswordController.php

use EntityManager;
use Illuminate\Foundation\Auth\ResetsPasswords;
use Illuminate\Support\Str;
...

    protected function resetPassword($user, $password)
    {
        $user->setPassword(bcrypt($password));
        $user->setRememberToken(Str::random(60));

        EntityManager::persist($user);
        EntityManager::flush();

        $this->guard()->login($user);
    }

```

Lastly... fix the `app.blade.php` file so it uses the getter for your name.

```html
<!-- app.blade.php -->
   <a href="#" class="dropdown-toggle" data-toggle="dropdown" role="button" aria-expanded="false">
        {{ Auth::user()->getName() }} <span class="caret"></span>
   </a>
```
