Entities
########

.. php:namespace:: Cake\ORM

.. php:class:: Entity

While :doc:`/orm/table-objects` represent and provide access to a collection of
objects, entities represent individual rows or domain objects in your
application. Entities contain methods to manipulate and
access the data they contain. Fields can also be accessed as properties on the object.

Entities are created for you each time you iterate the query instance returned
by ``find()`` of a table object or when you call ``all()`` or ``first()`` method
of the query instance.

Creating Entity Classes
=======================

You don't need to create entity classes to get started with the ORM in CakePHP.
However, if you want to have custom logic in your entities you will need to
create classes. By convention entity classes live in **src/Model/Entity/**. If
our application had an ``articles`` table we could create the following entity::

    // src/Model/Entity/Article.php
    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class Article extends Entity
    {
    }

Right now this entity doesn't do very much. However, when we load data from our
articles table, we'll get instances of this class.

.. note::

    If you don't define an entity class CakePHP will use the basic Entity class.

Creating Entities
=================

Entities can be directly instantiated::

    use App\Model\Entity\Article;

    $article = new Article();

When instantiating an entity you can pass the fields with the data you want
to store in them::

    use App\Model\Entity\Article;

    $article = new Article([
        'id' => 1,
        'title' => 'New Article',
        'created' => new DateTime('now')
    ]);

The preferred way of getting new entities is using the ``newEmptyEntity()`` method from the
``Table`` objects::

    use Cake\ORM\Locator\LocatorAwareTrait;

    $article = $this->fetchTable('Articles')->newEmptyEntity();

    $article = $this->fetchTable('Articles')->newEntity([
        'id' => 1,
        'title' => 'New Article',
        'created' => new DateTime('now')
    ]);

``$article`` will be an instance of ``App\Model\Entity\Article`` or fallback to
``Cake\ORM\Entity`` instance if you haven't created the ``Article`` class.

.. note::

    Prior to CakePHP 4.3 you need to use ``$this->getTableLocator->get('Articles')``
    to get the table instance.

Accessing Entity Data
=====================

Entities provide a few ways to access the data they contain. Most commonly you
will access the data in an entity using object notation::

    use App\Model\Entity\Article;

    $article = new Article;
    $article->title = 'This is my first post';
    echo $article->title;

You can also use the ``get()`` and ``set()`` methods::

    $article->set('title', 'This is my first post');
    echo $article->get('title');

When using ``set()`` you can update multiple fields at once using an array::

    $article->set([
        'title' => 'My first post',
        'body' => 'It is the best ever!'
    ]);

.. warning::

    When updating entities with request data you should configure which fields
    can be set with mass assignment.

You can check if fields are defined in your entities with ``has()``::

    $article = new Article([
        'title' => 'First post',
        'user_id' => null
    ]);
    $article->has('title'); // true
    $article->has('user_id'); // false
    $article->has('undefined'); // false.

The ``has()`` method will return ``true`` if a field is defined and has
a non-null value. You can use ``isEmpty()`` and ``hasValue()`` to check if
a field contains a 'non-empty' value::

    $article = new Article([
        'title' => 'First post',
        'user_id' => null
    ]);
    $article->isEmpty('title');  // false
    $article->hasValue('title'); // true

    $article->isEmpty('user_id');  // true
    $article->hasValue('user_id'); // false

Accessors & Mutators
====================

In addition to the simple get/set interface, entities allow you to provide
accessors and mutator methods. These methods let you customize how fields
are read or set.

Accessors use the convention of ``_get`` followed by the CamelCased version of
the field name.

.. php:method:: get($field)

They receive the basic value stored in the ``_fields`` array
as their only argument. Accessors will be used when saving entities, so be
careful when defining methods that format data, as the formatted data will be
persisted. For example::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class Article extends Entity
    {
        protected function _getTitle($title)
        {
            return ucwords($title);
        }
    }

The accessor would be run when getting the field through any of these two ways::

    echo $article->title;
    echo $article->get('title');

.. note::

    Code in your accessors is executed each time you reference the field. You can
    use a local variable to cache it if you are performing a resource-intensive
    operation in your accessor like this: `$myEntityProp = $entity->my_property`.

You can customize how fields get set by defining a mutator:

.. php:method:: set($field = null, $value = null)

Mutator methods should always return the value that should be stored in the
field. As you can see above, you can also use mutators to set other
calculated fields. When doing this, be careful to not introduce any loops,
as CakePHP will not prevent infinitely looping mutator methods.

Mutators allow you to convert fields as they are set, or create calculated
data. Mutators and accessors are applied when fields are read using property
access, or using ``get()`` and ``set()``. For example::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;
    use Cake\Utility\Text;

    class Article extends Entity
    {
        protected function _setTitle($title)
        {
            return Text::slug($title);
        }
    }

The mutator would be run when setting the field through any of these two
ways::

    $user->title = 'foo'; // slug is set as well
    $user->set('title', 'foo'); // slug is set as well

.. warning::

  Accessors are also run before entities are persisted to the database.
  If you want to transform fields but not persist that transformation,
  we recommend using virtual fields as those are not persisted.

.. _entities-virtual-fields:

Creating Virtual Fields
-----------------------

By defining accessors you can provide access to fields that do not
actually exist. For example if your users table has ``first_name`` and
``last_name`` you could create a method for the full name::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class User extends Entity
    {
        protected function _getFullName()
        {
            return $this->first_name . '  ' . $this->last_name;
        }
    }

You can access virtual fields as if they existed on the entity. The property
name will be the lower case and underscored version of the method (``full_name``)::

    echo $user->full_name;

Do bear in mind that virtual fields cannot be used in finds. If you want
them to be part of JSON or array representations of your entities,
see :ref:`exposing-virtual-fields`.

Checking if an Entity Has Been Modified
=======================================

.. php:method:: dirty($field = null, $dirty = null)

You may want to make code conditional based on whether or not fields have
changed in an entity. For example, you may only want to validate fields when
they change::

    // See if the title has been modified.
    $article->isDirty('title');

You can also flag fields as being modified. This is handy when appending into
array fields as this wouldn't automatically mark the field as dirty, only
exchanging completely would.::

    // Add a comment and mark the field as changed.
    $article->comments[] = $newComment;
    $article->setDirty('comments', true);

In addition you can also base your conditional code on the original field
values by using the ``getOriginal()`` method. This method will either return
the original value of the field if it has been modified or its actual value.

You can also check for changes to any field in the entity::

    // See if the entity has changed
    $article->isDirty();

To remove the dirty mark from fields in an entity, you can use the ``clean()``
method::

    $article->clean();

When creating a new entity, you can avoid the fields from being marked as dirty
by passing an extra option::

    $article = new Article(['title' => 'New Article'], ['markClean' => true]);

To get a list of all dirty fields of an ``Entity`` you may call::

    $dirtyFields = $entity->getDirty();

Validation Errors
=================

After you :ref:`save an entity <saving-entities>` any validation errors will be
stored on the entity itself. You can access any validation errors using the
``getErrors()``, ``getError()`` or ``hasErrors()`` methods::

    // Get all the errors
    $errors = $user->getErrors();

    // Get the errors for a single field.
    $errors = $user->getError('password');

    // Does the entity or any nested entity have an error.
    $user->hasErrors();

    // Does only the root entity have an error
    $user->hasErrors(false);

The ``setErrors()`` or ``setError()`` method can also be used to set the errors
on an entity, making it easier to test code that works with error messages::

    $user->setError('password', ['Password is required']);
    $user->setErrors([
        'password' => ['Password is required'],
        'username' => ['Username is required']
    ]);

.. _entities-mass-assignment:

Mass Assignment
===============

While setting fields to entities in bulk is simple and convenient, it can
create significant security issues. Bulk assigning user data from the request
into an entity allows the user to modify any and all columns. When using
anonymous entity classes or creating the entity class with the :doc:`/bake`
CakePHP does not protect against mass-assignment.

The ``_accessible`` property allows you to provide a map of fields and
whether or not they can be mass-assigned. The values ``true`` and ``false``
indicate whether a field can or cannot be mass-assigned::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class Article extends Entity
    {
        protected $_accessible = [
            'title' => true,
            'body' => true
        ];
    }

In addition to concrete fields there is a special ``*`` field which defines the
fallback behavior if a field is not specifically named::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class Article extends Entity
    {
        protected $_accessible = [
            'title' => true,
            'body' => true,
            '*' => false,
        ];
    }

.. note:: If the ``*`` field is not defined it will default to ``false``.

Avoiding Mass Assignment Protection
-----------------------------------

When creating a new entity using the ``new`` keyword you can tell it to not
protect itself against mass assignment::

    use App\Model\Entity\Article;

    $article = new Article(['id' => 1, 'title' => 'Foo'], ['guard' => false]);

Modifying the Guarded Fields at Runtime
---------------------------------------

You can modify the list of guarded fields at runtime using the ``setAccess()``
method::

    // Make user_id accessible.
    $article->setAccess('user_id', true);

    // Make title guarded.
    $article->setAccess('title', false);

.. note::

    Modifying accessible fields affects only the instance the method is called
    on.

When using the ``newEntity()`` and ``patchEntity()`` methods in the ``Table``
objects you can customize mass assignment protection with options. Please refer
to the :ref:`changing-accessible-fields` section for more information.

Bypassing Field Guarding
------------------------

There are some situations when you want to allow mass-assignment to guarded
fields::

    $article->set($fields, ['guard' => false]);

By setting the ``guard`` option to ``false``, you can ignore the accessible
field list for a single call to ``set()``.

Checking if an Entity was Persisted
-----------------------------------

It is often necessary to know if an entity represents a row that is already
in the database. In those situations use the ``isNew()`` method::

    if (!$article->isNew()) {
        echo 'This article was saved already!';
    }

If you are certain that an entity has already been persisted, you can use
``setNew()``::

    $article->setNew(false);

    $article->setNew(true);

.. _lazy-load-associations:

Lazy Loading Associations
=========================

While eager loading associations is generally the most efficient way to access
your associations, there may be times when you need to lazily load associated
data. Before we get into how to lazy load associations, we should discuss the
differences between eager loading and lazy loading associations:

Eager loading
    Eager loading uses joins (where possible) to fetch data from the
    database in as *few* queries as possible. When a separate query is required,
    like in the case of a HasMany association, a single query is emitted to
    fetch *all* the associated data for the current set of objects.
Lazy loading
    Lazy loading defers loading association data until it is absolutely
    required. While this can save CPU time because possibly unused data is not
    hydrated into objects, it can result in many more queries being emitted to
    the database. For example looping over a set of articles & their comments
    will frequently emit N queries where N is the number of articles being
    iterated.

While lazy loading is not included by CakePHP's ORM, you can just use one of the
community plugins to do so. We recommend `the LazyLoad Plugin
<https://github.com/jeremyharris/cakephp-lazyload>`__

After adding the plugin to your entity, you will be able to do the following::

    $article = $this->Articles->findById($id);

    // The comments property was lazy loaded
    foreach ($article->comments as $comment) {
        echo $comment->body;
    }

Creating Re-usable Code with Traits
===================================

You may find yourself needing the same logic in multiple entity classes. PHP's
traits are a great fit for this. You can put your application's traits in
**src/Model/Entity**. By convention traits in CakePHP are suffixed with
``Trait`` so they can be discernible from classes or interfaces. Traits are
often a good complement to behaviors, allowing you to provide functionality for
the table and entity objects.

For example if we had SoftDeletable plugin, it could provide a trait. This trait
could give methods for marking entities as 'deleted', the method ``softDelete``
could be provided by a trait::

    // SoftDelete/Model/Entity/SoftDeleteTrait.php

    namespace SoftDelete\Model\Entity;

    trait SoftDeleteTrait
    {
        public function softDelete()
        {
            $this->set('deleted', true);
        }
    }

You could then use this trait in your entity class by importing it and including
it::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;
    use SoftDelete\Model\Entity\SoftDeleteTrait;

    class Article extends Entity
    {
        use SoftDeleteTrait;
    }

Converting to Arrays/JSON
=========================

When building APIs, you may often need to convert entities into arrays or JSON
data. CakePHP makes this simple::

    // Get an array.
    // Associations will be converted with toArray() as well.
    $array = $user->toArray();

    // Convert to JSON
    // Associations will be converted with jsonSerialize hook as well.
    $json = json_encode($user);

When converting an entity to an JSON, the virtual & hidden field lists are
applied. Entities are recursively converted to JSON as well. This means that if you
eager loaded entities and their associations CakePHP will correctly handle
converting the associated data into the correct format.

.. _exposing-virtual-fields:

Exposing Virtual Fields
-----------------------

By default virtual fields are not exported when converting entities to
arrays or JSON. In order to expose virtual fields you need to make them
visible. When defining your entity class you can provide a list of virtual
field that should be exposed::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class User extends Entity
    {
        protected $_virtual = ['full_name'];
    }

This list can be modified at runtime using the ``setVirtual()`` method::

    $user->setVirtual(['full_name', 'is_admin']);

Hiding Fields
-------------

There are often fields you do not want exported in JSON or array formats. For
example it is often unwise to expose password hashes or account recovery
questions. When defining an entity class, define which fields should be
hidden::

    namespace App\Model\Entity;

    use Cake\ORM\Entity;

    class User extends Entity
    {
        protected $_hidden = ['password'];
    }

This list can be modified at runtime using the ``setHidden()`` method::

    $user->setHidden(['password', 'recovery_question']);

Storing Complex Types
=====================

Accessor & Mutator methods on entities are not intended to contain the logic for
serializing and unserializing complex data coming from the database. Refer to
the :ref:`saving-complex-types` section to understand how your application can
store more complex data types like arrays and objects.

.. meta::
    :title lang=en: Entities
    :keywords lang=en: entity, entities, single row, individual record
