# Ember-m3 [![Build Status](https://secure.travis-ci.org/hjdivad/ember-m3.svg?branch=master)](http://travis-ci.org/hjdivad/ember-m3)

This addon provides an alternative model implementation to `DS.Model` that is
compatible with the rest of the [ember-data](https://github.com/emberjs/data) ecosystem.

## Background

Ember-data users define their schemas via `DS.Model` classes which explicitly
state what attributes and relationships they expect.  Having many such classes
each explicitly defining their schemas provides a lot of clarity and a pleasant
environment for implementing standard object oriented principles.

However, it can be an issue in environments where the API responses are not
easily known in advance, or where they are so varied as to require thousands of
`DS.Model`s which can be a burden both to developer ergonomics as well as
runtime performance.

---

ember-m3 lets you use a single class for many API endpoints, inferring the
schema from the payload and API-specific conventions.

For example, if your API returns responses like the following:

```json
{
  "data": {
    "id": "isbn:9780439708180",
    "type": "com.example.bookstore.Book",
    "attributes": {
      "name": "Harry Potter and the Sorcerer's Stone",
      "author": "urn:Author:3",
      "chapters": [{
        "name": "The Boy Who Lived",
        "mentionedCharacters": ["urn:Character:harry"],
        "readerComments": [{
          "id": "urn:ReaderComment:1",
          "type": "com.example.bookstore.ReaderComment",
          "name": "Someone or Other",
          "body": "I have it on good authority that this is part of a book of some kind",
        }]
      }],
    },
  },
  "included": [{
    "id": "urn:author:3",
    "type": "com.example.bookstore.Author",
    "attributes": {
      "name": "JK Rowling",
    },
  }],
}
```

You could support it with the following schema:

```js
// app/initializers/schema-initializer.js
const BookstoreRegExp = /^com\.example\.bookstore\.*/;
const ISBNRegExp = /^isbn:.*/;
const URNRegExp = /^urn:(\w+):(.*)/;

export function initialize() {
  SchemaManager.registerSchema({
    includesModel(modelName) {
      return BookstoreRegExp.test(modelName);
    },

    computeAttributeReference(key, value) {
      if (!value) { return; }

      let match;

      if (ISBNRegExp.test(value)) {
        return {
          id: value,
          type: 'com.example.bookstore.Book',
        };
      } else if (match = URNRegExp.exec(value)) {
        return {
          id: match[2],
          type: `com.example.bookstore.${match[1]}`,
        }
      }
    },

    computeNestedModel(key, value) {
      if (value && typeof value === 'object') {
        return {
          id: value.id,
          type: value.type,
          attributes: value,
        }
      }
    },
  });
}

export default {
  name: 'm3-schema-initializer',
  initialize,
}
```

Notice that in this case, the schema doesn't specify anything model-specific and
would work whether the API returns 3 different kinds of models or 3,000.

Model-specific information *is* still needed to handle cases that cannot be
generally inferred from the payload (such as distinguishing `Date` fields).  See
the [Schema](#Schema) section for details.

## Trade-Offs

The benefits of using m3 over `DS.Model` are:

  - handle dynamic schemas whose structure is not known in advance
  - handle relationship references at arbitrary points in the payload seamlessly (eg relationship references within POJO attributes)
  - limit the payload size of schema information by inferring as much as
    possible from the structure of the payload itself
  - more easily query arbitrary URLs, especially when the returned models are
    not known in advance

The trade-offs made for this include:

  - Having only one model class prevents the use of some OOP patterns: You can't
    add computed properties to only one model for instance, and will need to
    rely on a different pattern of helpers and utility functions
  - Inferring the schema from the payload can make the client side code less
    clear as is often the case in "static" vs. "dynamic" tradeoffs

## Installation

* `ember install ember-m3`
* `ember generate schema-initializer`

## Querying

The existing store API works as expected. `findRecord`, `queryRecord` &c., will
build a URL using the `-ember-m3` adapter and create a record for the returned
response using `MegamorphicModel`.  Note that the actual name queried will be
passed to the adapter so you can build URLs correctly.

For example

```js
store.findRecord('com.example.bookstore.book', 'isbn:9780439708180');
```

Results in an adapter call

```js
import MegamorphicModel from 'ember-m3/model';

findRecord(store, modelClass, id, snapshot) {
  modelClass === MegamorphicModel;
  snapshot.modelName === 'com.example.bookstore.book';
  id === 'isbn:9780439708180';
}
```

Ember-m3 does not define an `-ember-m3` adapter but you can define one in your
app.  Otherwise the default adapter lookup rules are followed (ie your
`application` adapter will be used).

### Store.queryURL

ember-m3 also adds `store.queryURL`.  This is helpful for one-off endpoints or
endpoints where the type returned is not known and you just want a thin wrapper
around the API response that knows how to look up relationships.

```js
store.queryURL(url, options);
```

#### Return Value

Returns a promise that will resolve to

  1. A `MegamorphicModel` if the [primary data][json-api:primary-data] of the normalized response is a resource.
  2. A `RecordArray` of `MegamorphicModel`s if the [primary data][json-api:primary-data] of the normalized response is an array of
     resources.

The raw API response is normalized via the `-ember-m3` serializer.  M3 does not
define such a serializer but you can add one to your app if your API requires
normalization to JSON API.

#### Arguments

- `url` The URL path to query.  The `-ember-m3` adapter is consulted for its
  `host` and `namespace` properties.
  - When `url` is an absolute URL, (eg `http://bookstore.example.com/books`) or a network-path reference (eg `//books`), the adapter's `host` and `namespace` properties are ignored.
  - When `url` is an absolute path reference (eg `/books`) it is prefixed with the adapter's `host` and/or `namespace` if they are present.
  - When `url` is a relative path reference it is prefixed with the adapter's
    `host` and/or `namespace`, whichever is present.  It is an error to call
    `queryURL` when `url` is a relative path reference and the adapter specifies
    neither `host` nor `namespace`.

- `options` additional options.  All are optional, as is the `options` object
  itself.

  - `options.method` defaults to `GET`.  The HTTP method to use.

  - `options.params` defaults to `null`.  The parameters to include, either in the URL (for `GET`
    requests) or request body (for others).

  - `options.cacheKey` defaults to `null`.  A string to uniquely identify this request.  `null` or `undefined` indicates the result should not be cached.  It is passed to `serializer.normalizeResponse` as the `id` parameter, but the serializer is free to ignore it.

  - `options.reload` defaults to `false`.  If `true`, make a request even if an entry was found under
    `cacheKey`.  Do not resolve the returned promise until that request
    completes.

  - `options.backgroundReload`  defaults to `false`.  If `true`, make a request
    even if an entry was found under `cacheKey`.  If `true` and a cached entry
    was found, resolve the returned promise immediately with the cached entry
    and update the store when the request completes.

#### Caching

When `cacheKey` is provided the response payload is cached.  This can be useful
to show, eg dashboard data or any other data that changes over time.

Consider the following:

```js
store.queryURL('/newsfeed/latest', { cacheKey: 'newsfeed.latest', backgroundReload: true });
```

In this example, the first time the user visits a route that makes this query,
the promise will wait to resolve until the request completes.  The second time
the request is made the promise will resolve immediately with the cached values
while loading fresh values in the background.

Note that what is actually cached is the result: ie either a `MegamorphicModel`
or, more likely, a `RecordArray` of `MegamorphicModel`s.  

---

It is possible to do the same thing in stock Ember Data by making a `DS.Model`
class to wrap your search results and querying via:

```js
// app/models/news-feed.js
import DS from 'ember-data';
export DS.Model.extend({
  feedItems: DS.hasMany('feed-item'),
});

// somewhere, presumably in a route
store.findRecord('news-feed', 'latest', { backgroundReload: true });
```

As with ember-m3 generally, similar functionality is provided without the need
to create models and relationships within your app code.

##### Cache Eviction

Because models (or `RecordArray`s of models are cached) the cache can be emptied
automatically when the models are unloaded.  In the case of `RecordArray`s of
models, the entire cache entry is evicted if *any* of the member models is
unloaded.


## Schema

You have to register a schema to tell ember-m3 what types it should be enabled
for, as well as information that cannot be inferred from the response payload.
You can think of the schema as a single POJO that represents the same
information, more or less, as all of your `DS.Model` files.

You register your schema with a call to `SchemaManager.registerSchema(schema)`.
The `schema` you pass in is an object with the following properties.

- `includesModel(modelName)` Whether or not ember-m3 should handle this
  `modelName`.  It's fine to just `return true` here but this hook allows
  `ember-m3` to work alongside `DS.Model`.

- `computeAttributeReference(key, value)`  A function that determines
  whether an attribute is a reference.  If it is not, return `null` or
  `undefined`.
  Otherwise return an object with properties:
    - `id` The id of the referenced model (either m3 or `DS.Model`)
    - `type` The type of the referenced model (either m3 or `DS.Model`)

  Note that attribute references are all treated as synchronous.  There is no
  ember-m3 analogue to `DS.Model` async relationships.

-  `isAttributeArrayReference(key, value, modelName)` Whether the attribute
   should be treated as an array reference.  If `false` array values whose members are
   attribute references will still be resolved as an array of models.  If `true`
   they will be resoled as a `RecordArray` of models; additionally a
   `RecordArray` will be returned even for `null` values.

- `computeNestedModel(key, value, modelName)` Whether `value` should be treated
  as a nested model.  Useful for deeply nested references, eg with the following
  data:
  ```js
  {
    id: 1,
    type: 'com.example.library.book',
    attributes: {
      bestChapter: {
        number: 7,
        characterPOV: 'urn:character:2'
      }
    }
  }
  ```
  We would want `model.get('bestChapter.characterPOV')` to return the
  `character` model with id `2`, but this requires that the `bestChapter`
  attribute is treated as a nested m3 model and not a simple object.

  If `value` is a nested model, `computeNestedModel` must return an object with
  the properties `id`, `type` and `attributes`.  It is fine for this to simply
  treat all objects as nested models (be careful with transforms; you may
  want to explicitly check `value.constructor`).  eg
  ```js
  computeNestedModel(key, value, modelName) {
    if(value && value.constructor === Object) {
      return {
        id: value.id,
        type: value.type,
        attributes: value,
      };
    }
  }
  ```

- `models` an object containing type-specific information that cannot be
  inferred from the payload.  The `models` property has the form:
  ```js
  {
    models: {
      myModelName: {
        attributes: [],
        defaults: {
          attributeName: 'defaultValue',
        },
        aliases: {
          aliasName: 'attributeName',
        }
        transforms: {
          attributeName: transformFunctionn /* value */
        }
      }
    }
  }
  ```
  The keys to `models` are the types of your models, as they exist in your
  normalized payload.

  - `attributes` A list of whitelisted attributes.  It is recommended to omit
    this unless you explicitly want to prevent unknown properties returned in
    the API payload from being read.  If present, it is an array of strings that
    list whitelisted attributes.  Reads of non-whitelisted properties will
    return `undefined`.

  - `defaults` An object whose key-value pairs map attribute names to default
    values.  Reads of properties not included in the API will return the default
    value instead, if it is specified in the schema.

  - `aliases` Alternate names for payload attributes.  Aliases are read-only, ie
    equivalent to `Ember.computed.reads` and not `Ember.computed.alias`

  - `transforms` An object whose key-value pairs map attribute names to
    functions that transform their values.  This is useful to handle attributes
    that should be treated as `Date`s instead of strings, for instance.
    ```js
    function dateTransform(value) {
      if (!value) { return; }
      return new Date(Date.parse());
    }

    {
      models: {
        'com.example.bookstore.book': {
          transforms: {
            publishDate: dateTransform,
          }
        }
      }
    }
    ```

## Serializer / Adapter

Ember-m3 will use the `-ember-m3` adapter to make queries via `findRecord`,
`queryRecord, `queryURL` &c.  Responses will be normalized via the `-ember-m3`
serializer.

Ember-m3 provides neither an adapter nor a serializer.  If your app does not
define an `-ember-m3` adapter, the normal lookup rules are followed and your
`application` adapter is used instead 

It is perfectly fine to use your `application` adapter and serializer.  However,
if you have an app that uses both m3 models as well as `DS.Model`s you may
want to have different request headers, serialization or normalization for
your m3 models.  The `-ember-m3` adapter and serializer are the appropriate
places for this.

## Alternative Patterns

If you are converting an application that uses `DS.Model`s (perhaps because it
has a very large number of them and ember-m3 can help with performance) you may
have some patterns in your model classes beyond schema specification.

There are no particular requirements around refactoring these except that when
you only have a single class for your models you won't be able to use typical
object-oriented patterns.

The following are simply recommendations for common patterns.

### Constants

Use the schema `defaults` feature to replace constant values in your `DS.Model`
classes.  For example:

```js
// app/models/my-model.js

export DS.Model.extend({
  myConstant: 24601,
});

// convert to

// app/initializers/schema-initializer.js
{
  models: {
    'my-model': {
      defaults: {
        myConstant: 24601,
      }
    }
  }
}
```

### Ember.computed.reads

Use the schema `aliases` feature to replace use of `Ember.computed.reads`.  You
can likely do this also to replace the use of `Ember.computed.alias` as quite
often they can be read only.

```js
// app/models/my-model.js

export DS.Model.extend({
  name: DS.attr(),
  aliasName: Ember.computed.reads('name'),
});

// convert to

// app/initializers/schema-initializer.js
{
  models: {
    'my-model': {
      aliases: {
        aliasName: 'name',
      }
    }
  }
}
```


### Other Computed Properties

More involved computed properites can be converted to either utility functions
(if used within JavaScript) or helper functions (if used in templates).

For properties used in both templates and elsehwere (eg components) a convenient
pattern is to define a helper that exports both.


```js
// app/models/my-model.js
export DS.Model.extend({
  name: DS.attr('string'),
  sillyName: Ember.computed('name', function() {
    return `silly ${this.get('name')}`;
  }).readOnly(),
});
```
```hbs
{{! some-template.hbs }}
{{model.sillyName}}
{{my-component name=model.sillyName}}
```
```js
// app/routes/index.js
let sn = model.get('sillyName');
```

Coverted to

```js
// app/helpers/silly-name.js

export function getSillyName(model) {
  if (!model) { return; }
  return `silly ${model.get('name')}`;
};

function sillyNameHelper(positionalArgs) {
  if (positionalArgs.length < 1) {
    return;
  }

  return getSillyName(positionalArgs[0]);
}

export default Ember.Helper.helper(sillyNameHelper);
```
```hbs
{{! some-template.hbs }}
{{silly-name model}}
{{my-component name=(silly-name model)}}
```
```js
// app/routes/index.js
import { getSillyName } from '../helpers/silly-name'

// ...
let sn = getSillyName(model);
```

### Saving

Ember-m3 does not impose any particular requirements with saving models.  If
your endpoints cannot reliably be determined via `snapshot.modelName` it is
recommended to add support for `adapterOptions.url` in your adapter.  For
example:

```js
// app/adapters/-ember-m3.js
import ApplicationAdapter from './application';
export default ApplicationAdapter.extend({
  findRecord(store, type, id, snapshot) {
    let adapterOptions = snapshot.adapterOptions || {};
    let url = adapterOptions.url;
    if (!url) {
      url = this.buildURL(snapshot.modelName, id, snapshot, 'findRecord');
    }

    return this.ajax(url, 'GET');
  },

  // &c.
});


// somewhere else, perhaps in a route
this.store.findRecord('com.example.bookstore.book', 1, { url: '/book/from/surprising/endpoint' });
```


## Requirements

* ember@^2.12.0
* ember-data@~2.14.10

## Contributing

### Installation

* `git clone <repository-url>` this repository
* `cd ember-m3`
* `yarn install`

### Running Tests

* `yarn run test` (Runs `ember try:each` to test your addon against multiple Ember versions)
* `ember test`
* `ember test --server`

### Building

* `ember build`

For more information on using ember-cli, visit [https://ember-cli.com/](https://ember-cli.com/).

[json-api:primary-data]: http://jsonapi.org/format/#document-top-level
