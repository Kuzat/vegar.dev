---
title: "Hosting a Laravel App with Turso on Forge"
description: "Hosting a simple Laravel application using Turso on Laravel Forge."
date: "2024-07-18"
---
# Introduction
After learning that Turso made a Laravel driver for LibSQL I just knew I had to try it. So in this short tutorial I will setup a simple Laravel example application with Breeze, using Turso and libsql as the database. I will then show how to prepare a server on Forge to be able to host the application. Specifically I will show how we can install the LibSQL extension for php.

# Preparing the Turso Database
Lets start by getting a Turso database ready for the example. Let's create it using the Turso CLI
```sh
turso db create turso-laravel-example
```
We should now have a fresh new database we can use for this example. Run the following commands to get the url and generate a token to access the database.
```sh
turso db show --url turso-laravel-example
turso db tokens create turso-laravel-example
```
Save these for later as we will need them in the .env file for our Laravel application.

# Preparing the Laravel Application
Since the point of this article is to explore how we host it and not the application it self. So it will be just a standard breeze Laravel template application.
Let's start by creating a new Laravel project.
```sh
composer create-project laravel/laravel turso-laravel-example
```
We now have a Laravel project named `turso-laravel-example` and can continue by adding breeze and installing it.
```sh
cd turso-laravel-example/
composer require laravel/breeze --dev
php artisan breeze:install
```
When installing breeze we can just use the default options as they don't really matter for this tutorial.
We should now have a example application. What is missing is to install the `turso-libsql-driver` and make some changes so that we can use the database we created earlier.
First step is to composer require the driver. [[1]](https://docs.turso.tech/sdk/php/guides/laravel)
```sh
composer require tursodatabase/turso-driver-laravel
```
Now we also need to do some more changes to our project code for it to work. We need to Add the `LibSQLDriverServiceProvider` class to `bootstrap/providers.php`

After that we need to add this configuration to the `connections` array in `config/database.php`
```php
'libsql' => [
    'driver' => 'libsql',
    'url' => 'file:' . env('DB_DATABASE', 'database.sqlite'),
    'authToken' => env('DB_AUTH_TOKEN', ''),
    'syncUrl' => env('DB_SYNC_URL', ''),
    'syncInterval' => env('DB_SYNC_INTERVAL', 5),
    'read_your_writes' => env('DB_READ_YOUR_WRITES', true),
    'encryptionKey' => env('DB_ENCRYPTION_KEY', ''),
    'remoteOnly' => env('DB_REMOTE_ONLY', false),
    'database' => null,
    'prefix' => '',
],
```
You could now also setup your .env file and install the libsql extension/driver binary locally to test. However, since it is not yet supported in Herd I decided to skip this for now.

Final step before setting up the server is to create a git repo and add all the changes. Then push it to a repository at Github. I will not show how to do this in this tutorial.


# Installing LibSQL PHP Extension on Forge Server
After the last section we should have a Github repository with our `turso-laravel-example` project.
Which means we are now ready to deploy it to a server.

Before we create a new server in Forge we should first create a recipe for installing the libsql extension.
Here is a simple bash script I made that should fetch the Turso libsql php extension. It then extracts it and sets the required `extension=` variable in the `php.ini` files before reloading the php-fpm to pickup the changes. You should check the [Release](https://github.com/tursodatabase/turso-client-php/releases) page on Github to find the libsql version and php version that will work for you.
```bash
PHP_VERSION="8.3"
TURSO_LIBSQL_VERSION="v1.2.6"
DIR_NAME="libsql_php-turso-php-extension-${TURSO_LIBSQL_VERSION}-php-${PHP_VERSION}-x86_64-unknown-linux-gnu"
ARCHIVE_NAME="${DIR_NAME}.tar.gz"

cd /home/forge

curl -L https://github.com/tursodatabase/turso-client-php/releases/download/turso-php-extension-${TURSO_LIBSQL_VERSION}/${ARCHIVE_NAME} -o $ARCHIVE_NAME

tar -xzvf $ARCHIVE_NAME

# Add the extension to the php ini files
echo "extension=/home/forge/${DIR_NAME}/liblibsql_php.so" >> /etc/php/${PHP_VERSION}/fpm/php.ini
echo "extension=/home/forge/${DIR_NAME}/liblibsql_php.so" >> /etc/php/${PHP_VERSION}/cli/php.ini

service php${PHP_VERSION}-fpm reload
```
If you save it as a Recipe in Forge you can then use it when creating a server.
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/t7zgi0ek2k7byznbitnx.png)

For server provider I am using Hetzner, but you are free to choose your own. Since it is just an example project I just run a App Server with the smallest size. In `Advanced Settings` I set `Database` as `None` as we are going to use Turso. Finally we can choose a `Post-Provision Recipe` to run after the server is setup. Here we select the recipe we created earlier.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xy80voueimyppivoz8j6.png)

# Deploying the Application to Forge
By this point we should have a Laravel application in Github and a Server in Forge with Turso LibSQL php extension.
Since this is just an example I will deploy it to the default site for the server in Forge. Choosing the Github repo and create the site.

Before we can deploy the site we need to do some changes. Firstly we should change the deployment script to also include a `npm ci` and `npm run build`. The full deployment script I use is:
```bash
cd /home/forge/default
git pull origin $FORGE_SITE_BRANCH

$FORGE_COMPOSER install --no-dev --no-interaction --prefer-dist --optimize-autoloader
npm ci
npm run build

( flock -w 10 9 || exit 1
    echo 'Restarting FPM...'; sudo -S service $FORGE_PHP_FPM reload ) 9>/tmp/fpmlock

if [ -f artisan ]; then
    $FORGE_PHP artisan migrate --force
fi
```

Then secondly we need to configure our application to use the Turso database we created earlier.
I set these environment variables in the .env file for the site.
```
DB_CONNECTION=libsql
DB_AUTH_TOKEN=<your-database-auth-token-from-turso>
DB_SYNC_URL=<your-database-url-from-turso>
DB_REMOTE_ONLY=true
```
I am not sure if the `DB_REMOTE_ONLY` is needed all the time, but at least I was not able to do the initial migration without having it set.

The site should now be ready to deploy. We can then access it using the IP address of the server.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/wmaeghyukephb8t3vo0k.png)

And you should now see the default Laravel Breeze welcome page. You can try register and login and it should work, using your Turso database. We should also see it in the analytics for the database

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/kxkzumztdazm78ero3iu.png)

# Conclusion
We should now have a working Laravel deployment on Forge using a Turso libsql database.

The experience was not the easiest, and I bumped into a few errors when following the Turso guide for Laravel. Like they tell you to use the provider `Turso\Driver\Laravel\LibSQLServiceProvider::class` but what I found was a provider called `Turso\Driver\Laravel\LibSQLDriverServiceProvider::class` which was easy enough to fix. There was also the need to use the `DB_REMOTE_ONLY=true` when deploying to Forge, which gave me not so useful error message.

The biggest thing stopping me from fully using this for my Laravel projects is that it is not supported in Herd yet. So, working with it locally is not as smooth as it could be.
