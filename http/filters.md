# Request Filters
Request filters is a model designed specially to handle user request parameters and populate the target entity (for example ORM or ODM). In other words, this maps form values to model fields. RequestFilter can define it's own set of getters, setters, accessors and validations for fields. RequestFilter utilizes functionality of
InputManager to populate it's fields.
> It may be best to read about DataEnties, ORM, ODM and Validations first.

## Scaffolding
You can create a new `RequestFilter` by using the command `create:request name`. With ORM and ODM entities, you can pre-specify fields to be grabbed from request. 
If you wish to generate a request for your existing database entity simply use key '-e'.

Let's look at some examples. First of all, we can define a database entity:

```php
class User extends Record
{
    /**
     * @var array
     */
    protected $fillable = [
        'name',
        'status'
    ];
    
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'id'      => 'primary',
        'name'    => 'string',
        'status'  => 'enum(active,blocked)'
    ];
}
```

Now using console command `create:request user -e user` we will get something like this:

```php
/**
 * @property string $name
 * @property string $status
 */
class UseryRequest extends RequestFilter
{
    /**
     * Input schema.
     *
     * @var array
     */
    protected $schema = [
        'name'    => 'data:name',
        'status'  => 'data:status'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name'    => 'trim',
        'status'  => 'trim'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => [
            'notEmpty',
            'string'
        ],
        'status'  => [
            'notEmpty',
            'string'
        ]
    ];

    /**
     * {@inheritdoc}.
     *
     * @param \Database\User $entity Entity to be populated with request data.
     * @return bool
     */
    public function populate(EntityInterface $entity)
    {
        if (!parent::populate($entity)) {
            return false;
        }

        return $entity->isValid();
    }
}
```

Let's check out an example of request usage in controller method and then walk through it's schema definition:

```php
public function createUser(UserRequest $request)
{
    $user = new User();
    if(!$request->populate($user)) {
        dump($request->getErrors());
    }
}
```

As you can see, we declared method dependency for our request. This will automatically allow access to InputManager and popuplate it's fields described in it's schema.

```php
protected $schema = [
    'name'    => 'data:name',
    'status'  => 'data:status'
];
```

Schema definition will include the target field name, it's source and origin name specified using dot notation. We can easily switch our request to read the values from query:

```php
protected $schema = [
    'name'    => 'query:name',
    'status'  => 'query:status'
];
```

In addition, we can specify the origin name using dot notation.

```php
protected $schema = [
    'name'    => 'query:user.name',  //user[status]
    'status'  => 'query:user.status' //user[status]
];
```

RequestFilter supports different sources you can use for definition.  Any method in `InputManager` can be used as source:
```php
protected $schema = [
  'name'   => 'post:name',           //identical to "data:name"
  'field'  => 'query:field',         
  'file'   => 'file:images.preview', //Will be represented by UploadedFile Interface
  'secure' => 'isSecure'             //Alias for InputManager->isSecure()
];
```

**Other sources you can use:** uri (UriInterface), path (PAGE URI PATH), method (HTTP METHOD), isSecure (bool), isAjax (bool), isJsonExpected (bool), remoteAddress (string), header:origin, data:origin, post:origin, query:origin, cookie:origin, file:origin, server:origin, attribute:origin.

## Populating target Entity
Once the request is generated, you can map it's values to approprate fields to populate some database entity. You can either do it manually via get/setFields or use request `populate` method which will copy fields from request to target entity and validate it to make sure that everything is ok.

You can put your custom logic into your populate method to handle more complex scenarious. For example, within file uploads. 

```php
/**
 * @property string                $name
 * @property UploadedFileInterface $image
 */
class UserRequest extends RequestFilter
{
    /**
     * Input schema.
     *
     * @var array
     */
    protected $schema = [
        'name'  => 'data:name',
        'image' => 'file:picture'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name'   => 'trim',
        'status' => 'trim',
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'  => [
            'notEmpty',
            'string'
        ],
        'image' => [
            'image::uploaded',
            'image::valid'
        ]
    ];

    /**
     * {@inheritdoc}.
     *
     * @param \Database\User $entity Entity to be populated with request data.
     * @return bool
     */
    public function populate(EntityInterface $entity)
    {
        if (!parent::populate($entity)) {
            return false;
        }

        //Doing something with image
        $this->image->moveTo('...');


        return $entity->isValid();
    }
}
```

## Errors
Every error generated by request or target entity will be mapped to it's origin name. 

```php
protected $schema = [
    'name'    => 'query:user.name'
];
```

Fox example, if the request within a given schema returns any errors related to "name" field `getErrors`, method will return the following structure:
```php
[
    'user' => [
        'name' => 'Error message.'
    ]
];
```