Upgrading Instructions for Yii Framework 2.0
============================================

This file contains the upgrade notes for Yii 2.0. These notes highlight changes that
could break your application when you upgrade Yii from one version to another.
Even though we try to ensure backwards compatibility (BC) as much as possible, sometimes
it is not possible or very complicated to avoid it and still create a good solution to
a problem. You may also want to check out the [versioning policy](https://github.com/yiisoft/yii2/blob/master/docs/internals/versions.md)
for further details.

Upgrading in general is as simple as updating your dependency in your composer.json and
running `composer update`. In a big application however there may be more things to consider,
which are explained in the following.

> Note: This document assumes you have composer [installed globally](https://www.yiiframework.com/doc-2.0/guide-start-installation.html#installing-composer)
so that you can run the `composer` command. If you have a `composer.phar` file inside of your project you need to
replace `composer` with `php composer.phar` in the following.

> Tip: Upgrading dependencies of a complex software project always comes at the risk of breaking something, so make sure
you have a backup (you should be doing this anyway ;) ).

In case you use [composer asset plugin](https://github.com/fxpio/composer-asset-plugin) instead of the currently recommended
[asset-packagist.org](https://asset-packagist.org) to install Bower and NPM assets, make sure it is upgraded to the latest version as well. To ensure best stability you should also upgrade composer in this step:

    composer self-update
    composer global require "fxp/composer-asset-plugin:^1.4.1" --no-plugins

The simple way to upgrade Yii, for example to version 2.0.10 (replace this with the version you want) will be running `composer require`:

    composer require "yiisoft/yii2:~2.0.10" --update-with-dependencies

This command will only upgrade Yii and its direct dependencies, if necessary. Without `--update-with-dependencies` the
upgrade might fail when the Yii version you chose has slightly different dependencies than the version you had before.
`composer require` will by default not update any other packages as a safety feature.

Another way to upgrade is to change the `composer.json` file to require the new Yii version and then
run `composer update` by specifying all packages that are allowed to be updated.

    composer update yiisoft/yii2 yiisoft/yii2-composer bower-asset/inputmask

The above command will only update the specified packages and leave the versions of all other dependencies intact.
This helps to update packages step by step without causing a lot of package version changes that might break in some way.
If you feel lucky you can of course update everything to the latest version by running `composer update` without
any restrictions.

After upgrading you should check whether your application still works as expected and no tests are broken.
See the following notes on which changes to consider when upgrading from one version to another.

> Note: The following upgrading instructions are cumulative. That is,
if you want to upgrade from version A to version C and there is
version B between A and C, you need to follow the instructions
for both A and B.

Upgrade from Yii 2.0.52
-----------------------

* `ErrorHandler::convertExceptionToError()` has been deprecated and will be removed in version 2.2.0.

  This method was deprecated due to `PHP 8.4` deprecating the use of `E_USER_ERROR` with `trigger_error()`.
  The framework now handles exceptions in `__toString()` methods more appropriately based on the PHP version.

  **Before (deprecated):**
  ```php
  public function __toString() {
      try {
          return $this->render();
      } catch (\Throwable $e) {
          ErrorHandler::convertExceptionToError($e);
          return '';
      }
  }
  ```

  **After (recommended):**
  ```php
  public function __toString() {
      try {
          return $this->render();
      } catch (\Throwable $e) {
          if (PHP_VERSION_ID < 70400) {
              trigger_error(ErrorHandler::convertExceptionToString($e), E_USER_ERROR);
              return '';
          }
          throw $e;
      }
  }
  ```

* There was a bug when loading fixtures into PostgreSQL database, the table sequences were not reset. If you used a work-around or if you depended on this behavior, you are advised to review your code.

Upgrade from Yii 2.0.51
-----------------------

* The function signature for `yii\web\Session::readSession()` and `yii\web\Session::gcSession()` have been changed.
  They now have the same return types as `\SessionHandlerInterface::read()` and `\SessionHandlerInterface::gc()` respectively.
  In case those methods have overwritten you will need to update your child classes accordingly.

Upgrade from Yii 2.0.50
-----------------------

* Correcting the behavior for `JSON` column type in `MariaDb`.

  Example usage of `JSON` column type in `db`:
  
  ```php
  <?php
  
  use yii\db\Schema;
  
  $db = Yii::$app->db;
  $command = $db->createCommand();
  
  // Create a table with a JSON column
  $command->createTable(
      'products',
      [
          'id' => Schema::TYPE_PK,
          'details' => Schema::TYPE_JSON,
      ],
  )->execute();
  
  // Insert a new product
  $command->insert(
      'products',
      [
          'details' => [
              'name' => 'apple',
              'price' => 100,
              'color' => 'blue',
              'size' => 'small',
          ],
      ],
  )->execute();
  
  // Read all products
  $records = $db->createCommand('SELECT * FROM products')->queryAll();
  ```
  
  Example usage of `JSON` column type in `ActiveRecord`:
  
  ```php
  <?php
  
  namespace app\model;
  
  use yii\db\ActiveRecord;
  
  class ProductModel extends ActiveRecord
  {
      public static function tableName()
      {
          return 'products';
      }
  
      public function rules()
      {
          return [
              [['details'], 'safe'],
          ];
      }
  }
  ```
  
  ```php
  <?php
  
  use app\model\ProductModel;
  
  // Create a new product
  $product = new ProductModel();
  
  // Set the product details
  $product->details = [
      'name' => 'windows',
      'color' => 'red',
      'price' => 200,
      'size' => 'large',
  ];
  
  // Save the product
  $product->save();
  
  // Read the first product
  $product = ProductModel::findOne(1);
  
  // Get the product details
  $details = $product->details;
  
  echo 'Name: ' . $details['name'];
  echo 'Color: ' . $details['color'];
  echo 'Size: ' . $details['size'];
  
  // Read all products with color red
  $products = ProductModel::find()
      ->where(new \yii\db\Expression('JSON_EXTRACT(details, "$.color") = :color', [':color' => 'red']))
      ->all();
  
  // Loop through all products
  foreach ($products as $product) {
      $details = $product->details;
      echo 'Name: ' . $details['name'];
      echo 'Color: ' . $details['color'];
      echo 'Size: ' . $details['size'];
  }
  ```

Upgrade from Yii 2.0.48
-----------------------

* Since Yii 2.0.49 the `yii\console\Controller::select()` function supports a default value and respects
  the `yii\console\Controller::$interactive` setting. Before the user was always prompted to select an option
  regardless of the `$interactive` setting. Now the `$default` value is automatically returned when `$interactive` is 
  `false`.
* The function signature for `yii\console\Controller::select()` and `yii\helpers\BaseConsole::select()` have changed.
  They now have an additional `$default = null` parameter. In case those methods are overwritten you will need to
  update your child classes accordingly.

Upgrade from Yii 2.0.46
-----------------------

* The default "scope" of the `yii\mutex\MysqlMutex` has changed, the name of the mutex now includes the name of the
  database by default. Before this change the "scope" of the `MysqlMutex` was "server wide".
  No matter which database was used, the mutex lock was acquired for the entire server, since version 2.0.47 
  the "scope" of the mutex will be limited to the database being used. 
  This might impact your application if …
    * The database name of the database connection specified in the `MysqlMutex::$db` property is set dynamically
      (or changes in any other way during your application execution):  
      Depending on your application you might want to set the `MysqlMutex::$keyPrefix` property (see below).
    * The database connection specified in the `MysqlMutex::$db` does not include a database name:  
      You must specify the `MysqlMutex::$keyPrefix` property (see below).
  
  If you need to specify/lock the "scope" of the `MysqlMutex` you can specify the `$keyPrefix` property.  
  For example in your application config:   
    ```php
    'mutex' => [
        'class' => 'yii\mutex\MysqlMutex',
        'db' => 'db',
        'keyPrefix' => 'myPrefix' // Your custom prefix which determines the "scope" of the mutex.
    ],
    ```
  > Warning: Even if you're not impacted by the aforementioned conditions and even if you specify the `$keyPrefix`,
    if you rely on a locked mutex during and/or across your application deployment
    (e.g. switching over in a live environment from an old version to a new version of your application)
    you will have to make sure any running process that acquired a lock finishes before switching over to the new 
    version of your application. A lock acquired before the deployment will *not* be mutually exclusive with a 
    lock acquired after the deployment (even if they have the same name). 

Upgrade from Yii 2.0.45
-----------------------

* Changes in `Inflector::camel2words()` introduced in 2.0.45 were reverted, so it works as in pre-2.0.45. If you need
  2.0.45 behavior, [introduce your own method](https://github.com/yiisoft/yii2/pull/19495/files).
* `yii\log\FileTarget::$rotateByCopy` is now deprecated and setting it to `false` has no effect since rotating of 
  the files is done only by copy.
* `yii\validators\UniqueValidator` and `yii\validators\ExistValidator`, when used on multiple attributes, now only
  generate an error on a single attribute. Previously, they would report a separate error on each attribute.
  Old behavior can be achieved by setting `'skipOnError' => false`, but this might have undesired side effects with
  additional validators on one of the target attributes.
  See [issue #19407](https://github.com/yiisoft/yii2/issues/19407)

Upgrade from Yii 2.0.44
-----------------------

* `yii\filters\PageCache::$cacheHeaders` now takes a case-sensitive list of header names since PageCache is no longer 
  storing the normalized (lowercase) versions of them so make sure this list is properly updated and your page cache 
  is recreated.

Upgrade from Yii 2.0.43
-----------------------

* `Json::encode()` can now handle zero-indexed objects in same way as `json_encode()` and keep them as objects. In order 
  to avoid breaking backwards compatibility this behavior could be enabled by a new option flag but is disabled by default.
  * Set `yii/helpers/Json::$keepObjectType = true` anywhere in your application code
  * Or configure json response formatter to enable it for all JSON responses:
      ```php
      'response' => [
        'formatters' => [
          \yii\web\Response::FORMAT_JSON => [
            'class' => 'yii\web\JsonResponseFormatter',
            'prettyPrint' => YII_DEBUG, // use "pretty" output in debug mode
            'keepObjectType' => true, // keep object type for zero-indexed objects
          ],
        ],
      ],
      ```
* `yii\caching\Cache::multiSet()` now uses the default cache duration (`yii\caching\Cache::$defaultDuration`) when no 
  duration is provided. A duration of 0 should be explicitly passed if items should not expire.
* Default value of `yii\console\controllers\MessageController::$translator` is updated to `['Yii::t', '\Yii::t']`, since 
  old value of `'Yii::t'` didn't match `\Yii::t` calls on PHP 8. If configuration file for "extract" command overrides 
  default value, update config file accordingly. See [issue #18941](https://github.com/yiisoft/yii2/issues/18941)

Upgrade from Yii 2.0.42
-----------------------

* `yii\base\ErrorHandler` does not expose the `$_SERVER` information implicitly anymore.
* The methods `phpTypecast()` and `dbTypecast()` of `yii\db\ColumnSchema` will no longer convert `$value` from `int` to 
  `string`, if database column type is `INTEGER UNSIGNED` or `BIGINT UNSIGNED`.
  * I.e. it affects update and insert queries. For example:
  ```php
  \Yii::$app->db->createCommand()->insert('{{some_table}}', ['int_unsigned_col' => 22])->execute();
  ```
  will execute next SQL:
  ```sql
  INSERT INTO `some_table` (`int_unsigned_col`) VALUES (22)
  ```
* Property `yii\db\ColumnSchemaBuilder::$categoryMap` has been removed in favor of getter/setter methods `getCategoryMap()` 
  and `setCategoryMap()`.

Upgrade from Yii 2.0.41
-----------------------

* `NumberValidator` (`number`, `double`, `integer`) does not allow values with leading or terminating (non-trimmed) 
  white spaces anymore. If your application expects non-trimmed values provided to this validator make sure to trim 
  them first (i.e. by using `trim` / `filter` "validators").

Upgrade from Yii 2.0.40
-----------------------

* The methods `getAuthKey()` and `validateAuthKey()` of `yii\web\IdentityInterface` are now also used to validate active
  sessions (previously these methods were only used for cookie-based login). If your identity class does not properly
  implement these methods yet, you should update it accordingly (an example can be found in the guide under
  `Security` -> `Authentication`). Alternatively, you can simply return `null` in the `getAuthKey()` method to keep
  the old behavior (that is, no validation of active sessions). Applications that change the underlying `authKey` of
  an authenticated identity, should now call `yii\web\User::switchIdentity()`, `yii\web\User::login()`
  or `yii\web\User::logout()` to recreate the active session with the new `authKey`.

Upgrade from Yii 2.0.39.3
-------------------------

* Priority of processing `yii\base\Arrayable`, and `JsonSerializable` data has been reversed (`Arrayable` data is checked
  first now) in `yii\base\Model`, and `yii\rest\Serializer`. If your application relies on the previous priority you need 
  to fix it manually based on the complexity of desired (de)serialization result.

Upgrade from Yii 2.0.38
-----------------------

* The storage structure of the file cache has been changed when you use `\yii\caching\FileCache::$keyPrefix`.
It is worth warming up the cache again if there is a logical dependency when working with the file cache.

* `yii\web\Session` now respects the 'session.use_strict_mode' ini directive.
  In case you use a custom `Session` class and have overwritten the `Session::openSession()` and/or 
  `Session::writeSession()` functions changes might be required:
  * When in strict mode the `openSession()` function should check if the requested session id exists
    (and mark it for forced regeneration if not).
    For example, the `DbSession` does this at the beginning of the function as follows:
    ```php
    if ($this->getUseStrictMode()) {
        $id = $this->getId();
        if (!$this->getReadQuery($id)->exists()) {
            //This session id does not exist, mark it for forced regeneration
            $this->_forceRegenerateId = $id;
        }
    }
    // ... normal function continues ...
    ``` 
  * When in strict mode the `writeSession()` function should ignore writing the session under the old id.
    For example, the `DbSession` does this at the beginning of the function as follows:
    ```php
    if ($this->getUseStrictMode() && $id === $this->_forceRegenerateId) {
        //Ignore write when forceRegenerate is active for this id
        return true;
    }
    // ... normal function continues ...
    ```
  > Note: The sample code above is specific for the `yii\web\DbSession` class.
    Make sure you use the correct implementation based on your parent class,
    e.g. `yii\web\CacheSession`, `yii\redis\Session`, `yii\mongodb\Session`, etc.
  
  > Note: In case your custom functions call their `parent` functions, there are probably no changes needed to your 
    code if those parents implement the `useStrictMode` checks.

  > Warning: in case `openSession()` and/or `writeSession()` functions do not implement the `useStrictMode` code
    the session could be stored under a malicious id without warning even if `useStrictMode` is enabled.

Upgrade from Yii 2.0.37
-----------------------

* Resolving DI references inside of arrays in dependencies was made optional and turned off by default. In order
  to turn it on, set `resolveArrays` of container instance to `true`.

Upgrade from Yii 2.0.36
-----------------------

* `yii\db\Exception::getCode()` now returns full PDO code that is SQLSTATE string. If you have relied on comparing code
  with an integer value, adjust your code.

Upgrade from Yii 2.0.35
-----------------------

* Inline validator signature has been updated with 4th parameter `current`:

  ```php
  /**
   * @param mixed $current the currently validated value of attribute
   */
  function ($attribute, $params, $validator, $current)
  ```

* Behavior of inline validator used as a rule of `EachValidator` has been changed - `$attribute` now refers to original
  model's attribute and not its temporary counterpart:
  
  ```php
  public $array_attribute = ['first', 'second'];

  public function rules()
  {
      return [
          ['array_attribute', 'each', 'rule' => ['customValidatingMethod']],
      ];
  }
  
  public function customValidatingMethod($attribute, $params, $validator, $current)
  {
      // $attribute === 'array_attribute' (as before)
  
      // now: $this->$attribute === ['first', 'second'] (on every iteration)
      // previously:
      // $this->$attribute === 'first' (on first iteration)
      // $this->$attribute === 'second' (on second iteration)
  
      // use now $current instead
      // $current === 'first' (on first iteration)
      // $current === 'second' (on second iteration)
  }
  ```
  
* `$this` in an inline validator defined as closure now refers to model instance. If you need to access the object registering
  the validator, pass its instance through use statement:
  
  ```php
  $registrar = $this;
  $validator = function($attribute, $params, $validator, $current) use ($registrar) {
      // ...
  }
  ```
  
* Validator closure callbacks should not be declared as static.

* If you have any controllers that override the `init()` method, make sure they are calling `parent::init()` at
  the beginning, as demonstrated in the [component guide](https://www.yiiframework.com/doc/guide/2.0/en/concept-components).

Upgrade from Yii 2.0.34
-----------------------

* `ExistValidator` used as a rule of `EachValidator` now requires providing `targetClass` explicitely and it's not possible to use it with `targetRelation` in
  that configuration.
  
  ```php
  public function rules()
  {
      return [
          ['attribute', 'each', 'rule' => ['exist', 'targetClass' => static::className(), 'targetAttribute' => 'id']],
      ];
  }
  ```

Upgrade from Yii 2.0.32
-----------------------

* `yii\helpers\ArrayHelper::filter` now correctly filters data when passing a filter with more than 2 "levels",
  e.g. `ArrayHelper::filter($myArray, ['A.B.C']`. Until Yii 2.0.32 all data after the 2nd level was returned,
  please see the following example:
  
  ```php
  $myArray = [
      'A' => 1,
      'B' => [
          'C' => 1,
          'D' => [
              'E' => 1,
              'F' => 2,
          ]
      ],
  ];
  ArrayHelper::filter($myArray, ['B.D.E']);
  ```
  
  Before Yii 2.0.33 this would return
  
  ```php
  [
      'B' => [
          'D' => [
              'E' => 1,
              'F' => 2, //Please note the unexpected inclusion of other elements
          ],
      ],
  ]
  ```

  Since Yii 2.0.33 this returns

  ```php
  [
      'B' => [
          'D' => [
              'E' => 1,
          ],
      ],
  ]
  ```
  
  Note: If you are only using up to 2 "levels" (e.g. `ArrayHelper::filter($myArray, ['A.B']`), this change has no impact.
  
* `UploadedFile` class `deleteTempFile()` and `isUploadedFile()` methods introduced in 2.0.32 were removed.

* Exception will be thrown if `UrlManager::$cache` configuration is incorrect (previously misconfiguration was silently 
  ignored and `UrlManager` continue to work without cache). Make sure that `UrlManager::$cache` is correctly configured 
  or set it to `null` to explicitly disable cache.

Upgrade from Yii 2.0.31
-----------------------

* `yii\filters\ContentNegotiator` now generates 406 'Not Acceptable' instead of 415 'Unsupported Media Type' on
  content-type negotiation fail.

Upgrade from Yii 2.0.30
-----------------------
* `yii\helpers\BaseInflector::slug()` now ensures there is no repeating $replacement string occurrences.
  In case you rely on Yii 2.0.16 - 2.0.30 behavior, consider replacing `Inflector` with your own implementation.
  
  
Upgrade from Yii 2.0.28
-----------------------

* `yii\helpers\Html::tag()` now generates boolean attributes
  [according to HTML specification](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#boolean-attribute).
  For `true` value attribute is present, for `false` value it is absent.  

Upgrade from Yii 2.0.20
-----------------------

* `yii\db\Query::select()` and `addSelect()` now normalize the format that columns are stored in when saving them 
  to `$this->select`, so code that works directly with that property may need to be modified.
  
  For the following code:
  
  ```php
  $a = $query->select('*');
  $b = $query->select('id, name');
  ```
  
  The value was stored as is i.e.
  
  ```php
  // a
  ['*']
  
  // b
  ['id', 'name']
  ``` 
  
  Now it is stored as
  
  ```php
  // a
  ['*' => '*']
  
  // b
  ['id' => 'id', 'name' => 'name']
  ```

Upgrade from Yii 2.0.16
-----------------------

* In case you have extended the `yii\web\DbSession` class you should check if your 
  custom implementation is compatible with the new `yii\web\DbSession::$fields` attribute.
  Especially when overriding the `yii\web\DbSession::writeSession($id, $data)` function.

Upgrade from Yii 2.0.15
-----------------------

* Updated dependency to `cebe/markdown` to version `1.2.x`.
  If you need stick with 1.1.x, you can specify that in your `composer.json` by
  adding the following line in the `require` section:

  ```json
  "cebe/markdown": "~1.1.0",
  ```
  
* `yii\mutex\Mutex::acquire()` no longer returns `true` if lock is already acquired by the same component in the same process.
  Make sure that you're not trying to acquire the same lock multiple times in a way that may create infinite loops, for example:
    
  ```php
  if (Yii::$app->mutex->acquire('test')) {
       while (!Yii::$app->mutex->acquire('test')) {
           // `Yii::$app->mutex->acquire('test')` will always return `false` here, since lock is already acquired
      }
  }
  ```
  
* Formatter methods `asInteger`, `asDecimal`, `asPercent`, and `asCurrency` are using now inner fallback methods to handle 
  very big number values to counter inner PHP casting and floating point number presentation issues. Make sure to provide 
  such values as string numbers.
  
* Active Record relations are now being reset when corresponding key fields are changed. If you have relied on the fact
  that relations are never reloaded you have to adjust your code.


Upgrade from Yii 2.0.14
-----------------------

* When hash format condition (array) is used in `yii\db\ActiveRecord::findOne()` and `findAll()`, the array keys (column names)
  are now limited to the table column names. This is to prevent SQL injection if input was not filtered properly.
  You should check all usages of `findOne()` and `findAll()` to ensure that input is filtered correctly.
  If you need to find models using different keys than the table columns, use `find()->where(...)` instead.

  It's not an issue in the default generated code though as ID is filtered by
  controller code:

  The following code examples are **not** affected by this issue (examples shown for `findOne()` are valid also for `findAll()`):

  ```php
  // yii\web\Controller ensures that $id is scalar
  public function actionView($id)
  {
      $model = Post::findOne($id);
      // ...
  }
  ```

  ```php
  // casting to (int) or (string) ensures no array can be injected (an exception will be thrown so this is not a good practise)
  $model = Post::findOne((int) Yii::$app->request->get('id'));
  ```

  ```php
  // explicitly specifying the colum to search, passing a scalar or array here will always result in finding a single record
  $model = Post::findOne(['id' => Yii::$app->request->get('id')]);
  ```

  The following code however **is vulnerable**, an attacker could inject an array with an arbitrary condition and even exploit SQL injection:

  ```php
  $model = Post::findOne(Yii::$app->request->get('id'));
  ```

  For the above example, the SQL injection part is fixed with the patches provided in this release, but an attacker may still be able to search
  records by different condition than a primary key search and violate your application business logic. So passing user input directly like this can cause problems and should be avoided.


Upgrade from Yii 2.0.13
-----------------------

* Constants `IPV6_ADDRESS_LENGTH`, `IPV4_ADDRESS_LENGTH` were moved from `yii\validators\IpValidator` to `yii\helpers\IpHelper`.
  If your application relies on these constants, make sure to update your code to follow the changes.

* `yii\base\Security::compareString()` is now throwing `yii\base\InvalidArgumentException` in case non-strings are compared.

* `yii\db\ExpressionInterface` has been introduced to represent a wider range of SQL expressions. In case you check for
  `instanceof yii\db\Expression` in your code, you might consider changing that to checking for the interface and use the newly
  introduced methods to retrieve the expression content.

* Added JSON support for PostgreSQL and MySQL as well as Arrays support for PostgreSQL in ActiveRecord layer.
  In case you already implemented such support yourself, please switch to Yii implementation.
  * For MySQL JSON and PgSQL JSON & JSONB columns Active Record will return decoded JSON (that can be either array or scalar) after data population
  and expects arrays or scalars to be assigned for further saving them into a database.
  * For PgSQL Array columns Active Record will return `yii\db\ArrayExpression` object that acts as an array
  (it implements `ArrayAccess`, `Traversable` and `Countable` interfaces) and expects array or `yii\db\ArrayExpression` to be
  assigned for further saving it into the database.

  In case this change makes the upgrade process to Yii 2.0.14 too hard in your project, you can [switch off the described behavior](https://github.com/yiisoft/yii2/issues/15716#issuecomment-368143206)
  Then you can take your time to change your code and then re-enable arrays or JSON support.

* `yii\db\PdoValue` class has been introduced to replace a special syntax that was used to declare PDO parameter type 
  when binding parameters to an SQL command, for example: `['value', \PDO::PARAM_STR]`.
  You should use `new PdoValue('value', \PDO::PARAM_STR)` instead. Old syntax will be removed in Yii 2.1.

* `yii\db\QueryBuilder::conditionBuilders` property and method-based condition builders are no longer used. 
  Class-based conditions and builders are introduced instead to provide more flexibility, extensibility and
  space to customization. In case you rely on that property or override any of default condition builders, follow the 
  special [guide article](https://www.yiiframework.com/doc-2.0/guide-db-query-builder.html#adding-custom-conditions-and-expressions)
  to update your code.

* Protected method `yii\db\ActiveQueryTrait::createModels()` does not apply indexes as defined in `indexBy` property anymore.  
  In case you override default ActiveQuery implementation and relied on that behavior, call `yii\db\Query::populate()`
  method instead to index query results according to the `indexBy` parameter.

* Log targets (like `yii\log\EmailTarget`) are now throwing `yii\log\LogRuntimeException` in case log can not be properly exported.

* You can start preparing your application for Yii 2.1 by doing the following:

  - Replace `::className()` calls with `::class` (if you’re running PHP 5.5+).
  - Replace usages of `yii\base\InvalidParamException` with `yii\base\InvalidArgumentException`.
  - Replace calls to `Yii::trace()` with `Yii::debug()`.
  - Remove calls to `yii\BaseYii::powered()`.
  - If you are using XCache or Zend data cache, those are going away in 2.1 so you might want to start looking for an alternative.

* In case you aren't using CSRF cookies (REST APIs etc.) you should turn them off explicitly by setting
  `\yii\web\Request::$enableCsrfCookie` to `false` in your config file.
  
* Previously headers sent after content output was started were silently ignored. This behavior was changed to throwing
  `\yii\web\HeadersAlreadySentException`.

Upgrade from Yii 2.0.12
-----------------------

* The `yii\web\Request` class allowed to determine the value of `getIsSecureConnection()` form the
  `X-Forwarded-Proto` header if the connection was made via a normal HTTP request. This behavior
  was insecure as the header could have been set by a malicious client on a non-HTTPS connection.
  With 2.0.13 Yii adds support for configuring trusted proxies. If your application runs behind a reverse proxy and relies on
  `getIsSecureConnection()` to return the value form the `X-Forwarded-Proto` header you need to explicitly allow
  this in the Request configuration. See the [guide](https://www.yiiframework.com/doc-2.0/guide-runtime-requests.html#trusted-proxies) for more information.

  This setting also affects you when Yii is running on IIS webserver, which sets the `X-Rewrite-Url` header.
  This header is now filtered by default and must be listed in trusted hosts to be detected by Yii:

  ```php
  [   // accept X-Rewrite-Url from all hosts, as it will be set by IIS
      '/.*/' => ['X-Rewrite-Url'],
  ]
  ```

* For compatibiliy with [PHP 7.2 which does not allow classes to be named `Object` anymore](https://wiki.php.net/rfc/object-typehint),
  we needed to rename `yii\base\Object` to `yii\base\BaseObject`.
  
  `yii\base\Object` still exists for backwards compatibility and will be loaded if needed in projects that are
  running on PHP <7.2. The compatibility class `yii\base\Object` extends from `yii\base\BaseObject` so if you
  have classes that extend from `yii\base\Object` these would still work.
  
  What does not work however will be code that relies on `instanceof` checks or `is_subclass_of()` calls
  for `yii\base\Object` on framework classes as these do not extend `yii\base\Object` anymore but only
  extend from `yii\base\BaseObject`. In general such a check is not needed as there is a `yii\base\Configurable`
  interface you should check against instead.
  
  Here is a visualisation of the change (`a < b` means "b extends a"):
  
  ```
  Before:
  
  yii\base\Object < Framework Classes
  yii\base\Object < Application Classes
  
  After Upgrade:
  
  yii\base\BaseObject < Framework Classes
  yii\base\BaseObject < yii\base\Object < Application Classes

  ```
  
  If you want to upgrade PHP to version 7.2 in your project you need to remove all cases that extend `yii\base\Object`
  and extend from `yii\base\BaseObject` instead:
  
  ```
  yii\base\BaseObject < Framework Classes
  yii\base\BaseObject < Application Classes
  ```
  
  For extensions that have classes extending from `yii\base\Object`, to be compatible with PHP 7.2, you need to
  require `"yiisoft/yii2": "~2.0.13"` in composer.json and change affected classes to extend from `yii\base\BaseObject`
  instead. It is not possible to allow Yii versions `<2.0.13` and be compatible with PHP 7.2 or higher.

* A new method `public static function instance($refresh = false);` has been added to the `yii\db\ActiveRecordInterface` via a new
  `yii\base\StaticInstanceInterface`. This change may affect your application in the following ways:

  - If you have an `instance()` method defined in an `ActiveRecord` or `Model` class, you need to check whether the behavior is
    compatible with the method added by Yii.
  - Otherwise this method is implemented in the `yii\base\Model`, so the change only affects your code if you implement `ActiveRecordInterface`
    in a class that does not extend `Model`. You may use `yii\base\StaticInstanceTrait` to implement it.
    
* Fixed built-in validator creating when model has a method with the same name. 

  It is documented, that for the validation rules declared in model by `yii\base\Model::rules()`, validator can be either 
  a built-in validator name, a method name of the model class, an anonymous function, or a validator class name. 
  Before this change behavior was inconsistent with the documentation: method in the model had higher priority, than
  a built-in validator. In case you have relied on this behavior, make sure to fix it.

* Behavior was changed for methods `yii\base\Module::get()` and `yii\base\Module::has()` so in case when the requested
  component was not found in the current module, the parent ones will be checked for this component hierarchically.
  Considering that the root parent module is usually an application, this change can reduce calls to global `Yii::$app->get()`,
  and replace them with module-scope calls to `get()`, making code more reliable and easier to test.
  However, this change may affect your application if you have code that uses method `yii\base\Module::has()` in order
  to check existence of the component exactly in this specific module. In this case make sure the logic is not corrupted.

* If you are using "asset" command to compress assets and your web application `assetManager` has `linkAssets` turned on,
  make sure that "asset" command config has `linkAssets` turned on as well.


Upgrade from Yii 2.0.11
-----------------------

* `yii\i18n\Formatter::normalizeDatetimeValue()` returns now array with additional third boolean element
  indicating whether the timestamp has date information or it is just time value.

* `yii\grid\DataColumn` filter is now automatically generated as dropdown list with localized `Yes` and `No` strings
  in case of `format` being set to `boolean`.
 
* The signature of `yii\db\QueryBuilder::prepareInsertSelectSubQuery()` was changed. The method has got an extra optional parameter
  `$params`.

* The signature of `yii\cache\Cache::getOrSet()` has been adjusted to also accept a callable and not only `Closure`.
  If you extend this method, make sure to adjust your code.
  
* `yii\web\UrlManager` now checks if rules implement `getCreateUrlStatus()` method in order to decide whether to use
  internal cache for `createUrl()` calls. Ensure that all your custom rules implement this method in order to fully 
  benefit from the acceleration provided by this cache.

* `yii\filters\AccessControl` now can be used without `user` component. This has two consequences:

  1. If used without user component, `yii\filters\AccessControl::denyAccess()` throws `yii\web\ForbiddenHttpException` instead of redirecting to login page.
  2. If used without user component, using `AccessRule` matching a role throws `yii\base\InvalidConfigException`.
  
* Inputmask package name was changed from `jquery.inputmask` to `inputmask`. If you've configured path to
  assets manually, please adjust it. 

Upgrade from Yii 2.0.10
-----------------------

* A new method `public function emulateExecution($value = true);` has been added to the `yii\db\QueryInterace`.
  This method is implemented in the `yii\db\QueryTrait`, so this only affects your code if you implement QueryInterface
  in a class that does not use the trait.

* `yii\validators\FileValidator::getClientOptions()` and `yii\validators\ImageValidator::getClientOptions()` are now public.
  If you extend from these classes and override these methods, you must make them public as well.

* `yii\widgets\MaskedInput` inputmask dependency was updated to `~3.3.3`.
  [See its changelog for details](https://github.com/RobinHerbots/Inputmask/blob/3.x/CHANGELOG.md).

* PJAX: Auto generated IDs of the Pjax widget have been changed to use their own prefix to avoid conflicts.
  Auto generated IDs are now prefixed with `p` instead of `w`. This is defined by the `$autoIdPrefix`
  property of `yii\widgets\Pjax`. If you have any PHP or Javascript code that depends on autogenerated IDs
  you should update these to match this new value. It is not a good idea to rely on auto generated values anyway, so
  you better fix these cases by specifying an explicit ID.


Upgrade from Yii 2.0.9
----------------------

* RBAC: `getChildRoles()` method was added to `\yii\rbac\ManagerInterface`. If you've implemented your own RBAC manager
  you need to implement new method.

* Microsoft SQL `NTEXT` data type [was marked as deprecated](https://msdn.microsoft.com/en-us/library/ms187993.aspx) in MSSQL so
  `\yii\db\mssql\Schema::TYPE_TEXT` was changed from `'ntext'` to `'nvarchar(max)'

* Method `yii\web\Request::getBodyParams()` has been changed to pass full value of 'content-type' header to the second
  argument of `yii\web\RequestParserInterface::parse()`. If you create your own custom parser, which relies on `$contentType`
  argument, ensure to process it correctly as it may content additional data.

* `yii\rest\Serializer` has been changed to return a JSON array for collection data in all cases to be consistent among pages
  for data that is not indexed starting by 0. If your API relies on the Serializer to return data as JSON objects indexed by
  PHP array keys, you should set `yii\rest\Serializer::$preserveKeys` to `true`.


Upgrade from Yii 2.0.8
----------------------

* Part of code from `yii\web\User::loginByCookie()` method was moved to new `getIdentityAndDurationFromCookie()`
  and `removeIdentityCookie()` methods. If you override `loginByCookie()` method, update it in order use new methods.

* Fixture console command syntax was changed from `yii fixture "*" -User` to `yii fixture "*, -User"`. Upgrade your
  scripts if necessary.

Upgrade from Yii 2.0.7
----------------------

* The signature of `yii\helpers\BaseArrayHelper::index()` was changed. The method has got an extra optional parameter
  `$groups`.

* `yii\helpers\BaseArrayHelper` methods `isIn()` and `isSubset()` throw `\yii\base\InvalidParamException`
  instead of `\InvalidArgumentException`. If you wrap calls of these methods in try/catch block, change expected
  exception class.

* `yii\rbac\ManagerInterface::canAddChild()` method was added. If you have custom backend for RBAC you need to implement
  it.

* The signature of `yii\web\User::loginRequired()` was changed. The method has got an extra optional parameter
  `$checkAcceptHeader`.

* The signature of `yii\db\ColumnSchemaBuilder::__construct()` was changed. The method has got an extra optional
  parameter `$db`. In case you are instantiating this class yourself and using the `$config` parameter, you will need to
  move it to the right by one.

* String types in the MSSQL column schema map were upgraded to Unicode storage types. This will have no effect on
  existing columns, but any new columns you generate via the migrations engine will now store data as Unicode.

* Output buffering was introduced in the pair of `yii\widgets\ActiveForm::init()` and `::run()`. If you override any of
  these methods, make sure that output buffer handling is not corrupted. If you call the parent implementation, when
  overriding, everything should work fine. You should be doing that anyway.

Upgrade from Yii 2.0.6
----------------------

* Added new requirement: ICU Data version >= 49.1. Please, ensure that your environment has ICU data installed and
  up to date to prevent unexpected behavior or crashes. This may not be the case on older systems e.g. running Debian Wheezy.

  > Tip: Use Yii 2 Requirements checker for easy and fast check. Look for `requirements.php` in root of Basic and Advanced
  templates (howto-comment is in head of the script).

* The signature of `yii\helpers\BaseInflector::transliterate()` was changed. The method is now public and has an
  extra optional parameter `$transliterator`.

* In `yii\web\UrlRule` the `pattern` matching group names are being replaced with the placeholders on class
  initialization to support wider range of allowed characters. Because of this change:

  - You are required to flush your application cache to remove outdated `UrlRule` serialized objects.
    See the [Cache Flushing Guide](https://www.yiiframework.com/doc-2.0/guide-caching-data.html#cache-flushing)
  - If you implement `parseRequest()` or `createUrl()` and rely on parameter names, call `substitutePlaceholderNames()`
    in order to replace temporary IDs with parameter names after doing matching.

* The context of `yii.confirm` JavaScript function was changed from `yii` object to the DOM element which triggered
  the event.

  - If you overrode the `yii.confirm` function and accessed the `yii` object through `this`, you must access it
    with global variable `yii` instead.

* Traversable objects are now formatted as arrays in XML response to support SPL objects and Generators. Previous
  behavior could be turned on by setting `XmlResponseFormatter::$useTraversableAsArray` to `false`.

* If you've implemented `yii\rbac\ManagerInterface` you need to implement additional method `getUserIdsByRole($roleName)`.

* If you're using ApcCache with APCu, set `useApcu` to `true` in the component config.

* The `yii\behaviors\SluggableBehavior` class has been refactored to make it more reusable.
  Added new `protected` methods:

  - `isSlugNeeded()`
  - `makeUnique()`

  The visibility of the following Methods has changed from `private` to `protected`:

  - `validateSlug()`
  - `generateUniqueSlug()`

* The `yii\console\controllers\MessageController` class has been refactored to be better configurable and now also allows
  setting a lot of configuration options via command line. If you extend from this class, make sure it works as expected after
  these changes.

Upgrade from Yii 2.0.5
----------------------

* The signature of the following methods in `yii\console\controllers\MessageController` has changed. They have an extra parameter `$markUnused`.
  - `saveMessagesToDb($messages, $db, $sourceMessageTable, $messageTable, $removeUnused, $languages, $markUnused)`
  - `saveMessagesToPHP($messages, $dirName, $overwrite, $removeUnused, $sort, $markUnused)`
  - `saveMessagesCategoryToPHP($messages, $fileName, $overwrite, $removeUnused, $sort, $category, $markUnused)`
  - `saveMessagesToPO($messages, $dirName, $overwrite, $removeUnused, $sort, $catalog, $markUnused)`

Upgrade from Yii 2.0.4
----------------------

Upgrading from 2.0.4 to 2.0.5 does not require any changes.

Upgrade from Yii 2.0.3
----------------------

* Updated dependency to `cebe/markdown` to version `1.1.x`.
  If you need stick with 1.0.x, you can specify that in your `composer.json` by
  adding the following line in the `require` section:

  ```json
  "cebe/markdown": "~1.0.0",
  ```

Upgrade from Yii 2.0.2
----------------------

Starting from version 2.0.3 Yii `Security` component relies on OpenSSL crypto lib instead of Mcrypt. The reason is that
Mcrypt is abandoned and isn't maintained for years. Therefore your PHP should be compiled with OpenSSL support. Most
probably there's nothing to worry because it is quite typical.

If you've extended `yii\base\Security` to override any of the config constants you have to update your code:

    - `MCRYPT_CIPHER` — now encoded in `$cipher` (and hence `$allowedCiphers`).
    - `MCRYPT_MODE` — now encoded in `$cipher` (and hence `$allowedCiphers`).
    - `KEY_SIZE` — now encoded in `$cipher` (and hence `$allowedCiphers`).
    - `KDF_HASH` — now `$kdfHash`.
    - `MAC_HASH` — now `$macHash`.
    - `AUTH_KEY_INFO` — now `$authKeyInfo`.

Upgrade from Yii 2.0.0
----------------------

* Upgraded Twitter Bootstrap to [version 3.3.x](https://blog.getbootstrap.com/2014/10/29/bootstrap-3-3-0-released/).
  If you need to use an older version (i.e. stick with 3.2.x) you can specify that in your `composer.json` by
  adding the following line in the `require` section:

  ```json
  "bower-asset/bootstrap": "3.2.*",
  ```

Upgrade from Yii 2.0 RC
-----------------------

* If you've implemented `yii\rbac\ManagerInterface` you need to add implementation for new method `removeChildren()`.

* The input dates for datetime formatting are now assumed to be in UTC unless a timezone is explicitly given.
  Before, the timezone assumed for input dates was the default timezone set by PHP which is the same as `Yii::$app->timeZone`.
  This causes trouble because the formatter uses `Yii::$app->timeZone` as the default values for output so no timezone conversion
  was possible. If your timestamps are stored in the database without a timezone identifier you have to ensure they are in UTC or
  add a timezone identifier explicitly.

* `yii\bootstrap\Collapse` is now encoding labels by default. `encode` item option and global `encodeLabels` property were
 introduced to disable it. Keys are no longer used as labels. You need to remove keys and use `label` item option instead.

* The `yii\base\View::beforeRender()` and `yii\base\View::afterRender()` methods have two extra parameters `$viewFile`
  and `$params`. If you are overriding these methods, you should adjust the method signature accordingly.

* If you've used `asImage` formatter i.e. `Yii::$app->formatter->asImage($value, $alt);` you should change it
  to `Yii::$app->formatter->asImage($value, ['alt' => $alt]);`.

* Yii now requires `cebe/markdown` 1.0.0 or higher, which includes breaking changes in its internal API. If you extend the markdown class
  you need to update your implementation. See <https://github.com/cebe/markdown/releases/tag/1.0.0-rc> for details.
  If you just used the markdown helper class there is no need to change anything.

* If you are using CUBRID DBMS, make sure to use at least version 9.3.0 as the server and also as the PDO extension.
  Quoting of values is broken in prior versions and Yii has no reliable way to work around this issue.
  A workaround that may have worked before has been removed in this release because it was not reliable.

Upgrade from Yii 2.0 Beta
-------------------------

* If you are using Composer to upgrade Yii, you should run the following command first (once for all) to install
  the composer-asset-plugin, *before* you update your project:

  ```
  php composer.phar global require "fxp/composer-asset-plugin:~1.3.1"
  ```

  You also need to add the following code to your project's `composer.json` file:

  ```json
  "extra": {
      "asset-installer-paths": {
          "npm-asset-library": "vendor/npm",
          "bower-asset-library": "vendor/bower"
      }
  }
  ```

  It is also a good idea to upgrade composer itself to the latest version if you see any problems:

  ```
  composer self-update
  ```

* If you used `clearAll()` or `clearAllAssignments()` of `yii\rbac\DbManager`, you should replace
  them with `removeAll()` and `removeAllAssignments()` respectively.

* If you created RBAC rule classes, you should modify their `execute()` method by adding `$user`
  as the first parameter: `execute($user, $item, $params)`. The `$user` parameter represents
  the ID of the user currently being access checked. Previously, this is passed via `$params['user']`.

* If you override `yii\grid\DataColumn::getDataCellValue()` with visibility `protected` you have
  to change visibility to `public` as visibility of the base method has changed.

* If you have classes implementing `yii\web\IdentityInterface` (very common), you should modify
  the signature of `findIdentityByAccessToken()` as
  `public static function findIdentityByAccessToken($token, $type = null)`. The new `$type` parameter
  will contain the type information about the access token. For example, if you use
  `yii\filters\auth\HttpBearerAuth` authentication method, the value of this parameter will be
  `yii\filters\auth\HttpBearerAuth`. This allows you to differentiate access tokens taken by
  different authentication methods.

* If you are sharing the same cache across different applications, you should configure
  the `keyPrefix` property of the cache component to use some unique string.
  Previously, this property was automatically assigned with a unique string.

* If you are using `dropDownList()`, `listBox()`, `activeDropDownList()`, or `activeListBox()`
  of `yii\helpers\Html`, and your list options use multiple blank spaces to format and align
  option label texts, you need to specify the option `encodeSpaces` to be true.

* If you are using `yii\grid\GridView` and have configured a data column to use a PHP callable
  to return cell values (via `yii\grid\DataColumn::value`), you may need to adjust the signature
  of the callable to be `function ($model, $key, $index, $widget)`. The `$key` parameter was newly added
  in this release.

* `yii\console\controllers\AssetController` is now using hashes instead of timestamps. Replace all `{ts}` with `{hash}`.

* The database table of the `yii\log\DbTarget` now needs a `prefix` column to store context information.
  You can add it with `ALTER TABLE log ADD COLUMN prefix TEXT AFTER log_time;`.

* The `fileinfo` PHP extension is now required by Yii. If you use  `yii\helpers\FileHelper::getMimeType()`, make sure
  you have enabled this extension. This extension is [builtin](https://www.php.net/manual/en/fileinfo.installation.php) in php above `5.3`.

* Please update your main layout file by adding this line in the `<head>` section: `<?= Html::csrfMetaTags() ?>`.
  This change is needed because `yii\web\View` no longer automatically generates CSRF meta tags due to issue #3358.

* If your model code is using the `file` validation rule, you should rename its `types` option to `extensions`.

* `MailEvent` class has been moved to the `yii\mail` namespace. You have to adjust all references that may exist in your code.

* The behavior and signature of `ActiveRecord::afterSave()` has changed. `ActiveRecord::$isNewRecord` will now always be
  false in afterSave and also dirty attributes are not available. This change has been made to have a more consistent and
  expected behavior. The changed attributes are now available in the new parameter of afterSave() `$changedAttributes`.
  `$changedAttributes` contains the old values of attributes that had changed and were saved.

* `ActiveRecord::updateAttributes()` has been changed to not trigger events and not respect optimistic locking anymore to
  differentiate it more from calling `update(false)` and to ensure it can be used in `afterSave()` without triggering infinite
  loops.

* If you are developing RESTful APIs and using an authentication method such as `yii\filters\auth\HttpBasicAuth`,
  you should explicitly configure `yii\web\User::enableSession` in the application configuration to be false to avoid
  starting a session when authentication is performed. Previously this was done automatically by authentication method.

* `mail` component was renamed to `mailer`, `yii\log\EmailTarget::$mail` was renamed to `yii\log\EmailTarget::$mailer`.
  Please update all references in the code and config files.

* `yii\caching\GroupDependency` was renamed to `TagDependency`. You should create such a dependency using the code
  `new \yii\caching\TagDependency(['tags' => 'TagName'])`, where `TagName` is similar to the group name that you
  previously used.

* If you are using the constant `YII_PATH` in your code, you should rename it to `YII2_PATH` now.

* You must explicitly configure `yii\web\Request::cookieValidationKey` with a secret key. Previously this is done automatically.
  To do so, modify your application configuration like the following:

  ```php
  return [
      // ...
      'components' => [
          'request' => [
              'cookieValidationKey' => 'your secret key here',
          ],
      ],
  ];
  ```

  > Note: If you are using the `Advanced Project Template` you should not add this configuration to `common/config`
  or `console/config` because the console application doesn't have to deal with CSRF and uses its own request that
  doesn't have `cookieValidationKey` property.

* `yii\rbac\PhpManager` now stores data in three separate files instead of one. In order to convert old file to
new ones save the following code as `convert.php` that should be placed in the same directory your `rbac.php` is in:

  ```php
  <?php
  $oldFile = 'rbac.php';
  $itemsFile = 'items.php';
  $assignmentsFile = 'assignments.php';
  $rulesFile = 'rules.php';

  $oldData = include $oldFile;

  function saveToFile($data, $fileName) {
      $out = var_export($data, true);
      $out = "<?php\nreturn " . $out . ';';
      $out = str_replace(['array (', ')'], ['[', ']'], $out);
      file_put_contents($fileName, $out, LOCK_EX);
  }

  $items = [];
  $assignments = [];
  if (isset($oldData['items'])) {
      foreach ($oldData['items'] as $name => $data) {
          if (isset($data['assignments'])) {
              foreach ($data['assignments'] as $userId => $assignmentData) {
                  $assignments[$userId][] = $assignmentData['roleName'];
              }
              unset($data['assignments']);
          }
          $items[$name] = $data;
      }
  }

  $rules = [];
  if (isset($oldData['rules'])) {
      $rules = $oldData['rules'];
  }

  saveToFile($items, $itemsFile);
  saveToFile($assignments, $assignmentsFile);
  saveToFile($rules, $rulesFile);

  echo "Done!\n";
  ```

  Run it once, delete `rbac.php`. If you've configured `authFile` property, remove the line from config and instead
  configure `itemFile`, `assignmentFile` and `ruleFile`.

* Static helper `yii\helpers\Security` has been converted into an application component. You should change all usage of
  its methods to a new syntax, for example: instead of `yii\helpers\Security::hashData()` use `Yii::$app->getSecurity()->hashData()`.
  The `generateRandomKey()` method now produces not an ASCII compatible output. Use `generateRandomString()` instead.
  Default encryption and hash parameters has been upgraded. If you need to decrypt/validate data that was encrypted/hashed
  before, use the following configuration of the 'security' component:

  ```php
  return [
      'components' => [
          'security' => [
              'derivationIterations' => 1000,
          ],
          // ...
      ],
      // ...
  ];
  ```

* If you are using query caching, you should modify your relevant code as follows, as `beginCache()` and `endCache()` are
  replaced by `cache()`:

  ```php
  $db->cache(function ($db) {

     // ... SQL queries that need to use query caching

  }, $duration, $dependency);
  ```

* Due to significant changes to security you need to upgrade your code to use `\yii\base\Security` component instead of
  helper. If you have any data encrypted it should be re-encrypted. In order to do so you can use old security helper
  [as explained by @docsolver at github](https://github.com/yiisoft/yii2/issues/4461#issuecomment-50237807).

* [[yii\helpers\Url::to()]] will no longer prefix base URL to relative URLs. For example, `Url::to('images/logo.png')`
  will return `images/logo.png` directly. If you want a relative URL to be prefix with base URL, you should make use
  of the alias `@web`. For example, `Url::to('@web/images/logo.png')` will return `/BaseUrl/images/logo.png`.

* The following properties are now taking `false` instead of `null` for "don't use" case:
  - `yii\bootstrap\NavBar::$brandLabel`.
  - `yii\bootstrap\NavBar::$brandUrl`.
  - `yii\bootstrap\Modal::$closeButton`.
  - `yii\bootstrap\Modal::$toggleButton`.
  - `yii\bootstrap\Alert::$closeButton`.
  - `yii\widgets\LinkPager::$nextPageLabel`.
  - `yii\widgets\LinkPager::$prevPageLabel`.
  - `yii\widgets\LinkPager::$firstPageLabel`.
  - `yii\widgets\LinkPager::$lastPageLabel`.

* The format of the Faker fixture template is changed. For an example, please refer to the file
  `apps/advanced/common/tests/templates/fixtures/user.php`.

* The signature of all file downloading methods in `yii\web\Response` is changed, as summarized below:
  - `sendFile($filePath, $attachmentName = null, $options = [])`
  - `sendContentAsFile($content, $attachmentName, $options = [])`
  - `sendStreamAsFile($handle, $attachmentName, $options = [])`
  - `xSendFile($filePath, $attachmentName = null, $options = [])`

* The signature of callbacks used in `yii\base\ArrayableTrait::fields()` is changed from `function ($field, $model) {`
  to `function ($model, $field) {`.

* `Html::radio()`, `Html::checkbox()`, `Html::radioList()`, `Html::checkboxList()` no longer generate the container
  tag around each radio/checkbox when you specify labels for them. You should manually render such container tags,
  or set the `item` option for `Html::radioList()`, `Html::checkboxList()` to generate the container tags.

* The formatter class has been refactored to have only one class regardless whether PHP intl extension is installed or not.
  Functionality of `yii\base\Formatter` has been merged into `yii\i18n\Formatter` and `yii\base\Formatter` has been
  removed so you have to replace all usage of `yii\base\Formatter` with `yii\i18n\Formatter` in your code.
  Also the API of the Formatter class has changed in many ways.
  The signature of the following Methods has changed:

  - `asDate`
  - `asTime`
  - `asDatetime`
  - `asSize` has been split up into `asSize` and `asShortSize`
  - `asCurrency`
  - `asDecimal`
  - `asPercent`
  - `asScientific`

  The following methods have been removed, this also means that the corresponding format which may be used by a
  GridView or DetailView is not available anymore:

  - `asNumber`
  - `asDouble`

  Also due to these changes some formatting defaults have changes so you have to check all your GridView and DetailView
  configuration and make sure the formatting is displayed correctly.

  The configuration for `asSize()` has changed. It now uses the configuration for the number formatting from intl
  and only the base is configured using `$sizeFormatBase`.

  The specification of the date and time formats is now using the ICU pattern format even if PHP intl extension is not installed.
  You can prefix a date format with `php:` to use the old format of the PHP `date()`-function.

* The DateValidator has been refactored to use the same format as the Formatter class now (see previous change).
  When you use the DateValidator and did not specify a format it will now be what is configured in the formatter class instead of 'Y-m-d'.
  To get the old behavior of the DateValidator you have to set the format explicitly in your validation rule:

  ```php
  ['attributeName', 'date', 'format' => 'php:Y-m-d'],
  ```

* `beforeValidate()`, `beforeValidateAll()`, `afterValidate()`, `afterValidateAll()`, `ajaxBeforeSend()` and `ajaxComplete()`
  are removed from `ActiveForm`. The same functionality is now achieved via JavaScript event mechanism like the following:

  ```js
  $('#myform').on('beforeValidate', function (event, messages, deferreds) {
      // called when the validation is triggered by submitting the form
      // return false if you want to cancel the validation for the whole form
  }).on('beforeValidateAttribute', function (event, attribute, messages, deferreds) {
      // before validating an attribute
      // return false if you want to cancel the validation for the attribute
  }).on('afterValidateAttribute', function (event, attribute, messages) {
      // ...
  }).on('afterValidate', function (event, messages) {
      // ...
  }).on('beforeSubmit', function () {
      // after all validations have passed
      // you can do ajax form submission here
      // return false if you want to stop form submission
  });
  ```

* The signature of `View::registerJsFile()` and `View::registerCssFile()` has changed. The `$depends` and `$position`
  paramaters have been merged into `$options`. The new signatures are as follows:

  - `registerJsFile($url, $options = [], $key = null)`
  - `registerCssFile($url, $options = [], $key = null)`
