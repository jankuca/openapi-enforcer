
# OpenAPI-Enforcer

**Supports OpenAPI 2.0 (formerly Swagger) and OpenAPI 3.0.0**

Features

- Connect middleware*
- Request parsing and validating*
- Response building, formatting, and validating*
- Schema validation

\* *Some features coming soon*

**THIS IS A WORK IN PROGRESS - SUBJECT TO CHANGE**

# Table of Contents

- [Constructor](#constructor)
- Prototype Methods
    - [Enforcer.prototype.deserialize](#enforcerprototypedeserialize)
    - [Enforcer.prototype.errors](#enforcerprototypeerrors)
    - [Enforcer.prototype.path](#enforcerprototypepath)
    - [Enforcer.prototype.populate](#enforcerprototypepopulate)
    - [Enforcer.prototype.schema](#enforcerprototypeschema)
    - [Enforcer.prototype.serialize](#enforcerprototypeserialize)
    - [Enforcer.prototype.validate](#enforcerprototypevalidate)
- Static Methods
    - [Enforcer.format](#enforcerformat)
    - [Enforcer.is](#enforceris)
    - [Enforcer.parse](#enforcerparse)

# Example

```js
const RefParser = require('json-schema-ref-parser');
const Enforcer = require('openapi-enforcer');

// load, parse, dereference openapi document
RefParser.dereference('/path/to/schema/file.json')  // path can also be yaml
    .then(function(schema) {

        // create an enforcer instance
        const enforcer = Enforcer(schema);

        // get the schema that defines a user
        const userSchema = schema.components.schemas.user;

        // create a user object that uses the schema and variable mapping
        const user = enforcer.populate(userSchema, {
            name: 'Bob Smith',
            birthday: new Date('2000-01-01')
        });

        // check the user object for any schema errors
        // (FYI - it wont have errors because it was just populated from the schema)
        const errors = enforcer.errors(userSchema, user);

        // continue processing
    });
```

# Constructor

Create an OpenAPI enforcer instance.

`new Enforcer ( definition [, options ] )`

| Parameter | Description | Type | Default |
| --------- | ----------- | ---- | ------- |
| definition | An openapi document or a string representing the version to use. | `string` or `object` | |
| options | The configuration options to apply to the instance. | `object` | `{}` |

**Returns** an instance of the OpenAPI Enforcer

**Example 1 - Version as parameter**

```js
const Enforcer = require('openapi-enforcer');
const enforcer = new Enforcer('2.0');   // create an enforcer for OpenAPI version 2.0
```

**Example 2 - Object as parameter**

```js
const Enforcer = require('openapi-enforcer');
const enforcer = new Enforcer({ openapi: '3.0.0' });   // create an enforcer for OpenAPI version 3.0.0
```

### Options

The `options` object defines options for several functions. Those options are broken into their specific categories.

```js
const options = {
    populate: {     // options apply to enforcer.populate
        autoFormat: false,
        copy: false,
        defaults: true,
        ignoreMissingRequired: true,
        replacement: 'handlebar',
        templateDefaults: true,
        templates: true,
        variables: true
    },
    
    request: {      // options apply to enforce.middleware and enforce.request
        purge: true,
        strict: true
    },
    
    validate: {     // options apply to enforce.errors and enforce.validate
        
    }
}
```

#### options.populate.autoFormat

If set to `true` then values will automatically be [formatted](#enforcerformat--schema-value-) while populating.

Default: `false` 

#### options.populate.copy

When executing [`enforcer.populate(schema, params [, initialValue ])`](#enforcerpopulate--schema-params--value--) and providing an `initialValue` you have the option to either mutate (modify) that value or to create a copy of the value and mutate that. Mutation is faster, but if you do not want to change the passed in `initialValue` then you should set this value to `true`. 

Default: `false`

#### options.populate.defaults

Allow populated values to be built from a schema's `default` value. 

[More about default, x-template, and x-variable](#about-default-x-template-and-x-variable).

Default: `true`

#### options.populate.ignoreMissingRequired

When executing [`enforcer.populate(schema, params [, initialValue ])`](#enforcerpopulate--schema-params--value--) there will be times where an object with required properties is missing values for those required properties. If this value is set to `false` then [`enforcer.populate`](#enforcerpopulate--schema-params--value--) will not add the object to the populated value. If set to `true` then partially completed objects will still be added to the populated value.

Default: `true`

#### options.populate.replacement

The template [parameter replacement](#parameter-replacement) format to use. This can be one of `'handlebar'`, `'doubleHandlebar'`, or `'colon'`. 

| Format | Example |
| ------ | ------- |
| handlebar | `{param}` |
| doubleHandlebar | `{{param}}` |
| colon | `:param` |

Default: `'handlebar'`

#### options.populate.templateDefaults

If this is set to `true` and a default is being use to populate a value and the default value is a string then the value will act as an `x-template` value. This can be useful because `default` values generally appear in generated documentation but you may still want to perform an `x-template` transformation on the value.

#### options.populate.templates

Allow populated values to be built from a schema's `x-template` value. 

[More about default, x-template, and x-variable](#about-default-x-template-and-x-variable).

Default: `true`

#### options.populate.variables

Allow populated values to be built from a schema's `x-variable` value. 

[More about default, x-template, and x-variable](#about-default-x-template-and-x-variable).

Default: `true`

#### options.request.strict

If data is provided in the request body or query parameters that is not defined in the OpenAPI document then a 400 error will be generated.

Default: `false`

#### options.request.files

TODO: move this to request function - files and fields should be in body

The OpenAPI Enforcer does not support request body parsing. Use existing tools like [body-parser](https://www.npmjs.com/package/body-parser) or [formidable](https://www.npmjs.com/package/formidable) for that.

If a request's body has `multipart/form-data` content with binary file uploads then the data associated

If a request is sending one or more files then the request parser/validator needs to know where those files are being stored on the request object. Validation on 

```js
const Enforcer = require('openapi-enforcer');
const express = require('express');
const formidable = require('formidable');

const options = {
    request: { files: 'files' }
};
const enforcer = new Enforcer({ openapi: '3.0.0' }, options);

const app = express();

// middleware to parse multipart/form-data 
app.use(function (req, res, next) {
    const form = new formidable.IncomingForm();
    form.parse(req, (err, fields, files) => {
        if (err) return next(err);
        req.body = Object.assign({}, fields, files);
        next();
    });
});

app.use(enforcer.middleware());
```

Default: `'files'`

## Enforcer.prototype.deserialize

Convert a serialized value into its deserialized equivalent.

`Enforcer.prototype.deserialize ( schema, value )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| schema | The schema to use to convert serialized values. | `object` |
| value | The value to deserialize. | Any |

Returns: An object with two properties: `error` and `value`. If deserialization failed due to an error then the `error` property will contain details about the error. If deserialization succeeds then the `error` property will be `null` and the `value` property will have the deserialized value.

```js
const Enforcer = require('openapi-enforcer');

// create the enforcer instance
const enforcer = new Enforcer({ openapi: '3.0.0' });

// define a schema that defines deserialization instructions
const schema = {
    type: 'object',
    additionalProperties: false,
    properties: {
        integers: {
            type: 'array',
            items: {
                type: 'integer'
            }
        },
        date: {
            type: 'string',
            format: 'date-time'
        }
    }
};

const serializedValue = {
    integers: [1, '2', 3.1, '3.8'],
    date: '2000-01-01T01:02:03:456Z'
};

const deserialized = enforcer.deserialize(schema, serializedValue);
// {
//    error: null,
//    value: {
//        integers: [1, 2, 3, 4],
//        date: <Date Object>
//    }
// }
```

## Enforcer.prototype.errors

Validate a value against a schema and receive a detailed report where errors exist and why.

`Enforcer.prototype.errors ( schema, value )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| schema | The schema to validate the value against. | `object` |
| value | The value to validate. | Any |

Returns: An array of strings where each item in the array describes one error that was encountered.

```js
const Enforcer = require('openapi-enforcer');

// create the enforcer instance
const enforcer = new Enforcer({ openapi: '3.0.0' });

// define a schema to validate values against
const schema = {
    type: 'object',
    additionalProperties: false,
    properties: {
        names: {
            type: 'array',
            items: {
                type: 'string',
                minLength: 1
            }
        },
        date: {
            type: 'string',
            format: 'date-time',
            maximum: '2000-01-01T00:00:00.000Z'
        }
    }
};

// get any errors and log to console
const errors = enforcer.errors(schema, {
    names: [ 'Bob', 'Jan', '' ],
    date: '2010-01-01T00:00:00.000Z',
    num: 8
});
// errors ==> [
//   /names/2: String length below minimum length of 1 with length of 0: ''
//   /date: Expected date-time to be less than or equal to 2000-01-01T00:00:00.000Z. Received: 2010-01-01T00:00:00.000Z
//   /num: Property not allowed
// ]
```

## Enforcer.prototype.path

Get the matching path's path parameters and schema.

`Enforcer.prototype.path ( path )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| path | The lookup path (optionally with path parameters) | `string` |

Returns: An object with the `path` (definition path), `params` (path parameters), and `schema` definition path schema.

```js
const pathItem = {
    get: {},
    parameters: [
        {
            name: 'id',
            in: 'path',
            required: true,
            schema: {
                type: 'number'
            }
        }
    ]
};

const definition = {
    openapi: '3.0.0',
    paths: {
        '/path/{id}': pathItem
    }
}

// create the enforcer instance
const enforcer = new Enforcer(definition);

const match = enforcer.path('/path/25');
// {
//     path: '/path/{id}',
//     params: {
//         id: 25
//     },
//     schema: { ... } <== pathItem object
// }
```

## Enforcer.prototype.populate

Build a value from a schema. While traversing the schema the final populated value may be derived from the provided value in combination with the schema's `default` value, the `x-template` value, or the `x-variable` value.

`Enforcer.prototype.populate ( schema, params [, value ] )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| schema | The schema to build from | `object` |
| params | A map of keys to values. These values are used to help build the final value | `object` |
| value | An initial value to start with. | Any |

Returns: The populated value.

### About default, x-template, and x-variable

The `default` attribute is part of the OpenAPI specification. The type of it's value must be the same as the schema type. For example, if the schema is of type string, default cannot be a number. When `default` is a string [it can behave](#options-populate-templatedefaults) like `x-template` and [substitute parameters](#parameter-replacement) into the string. The advantage of using `default` over `x-template` in this scenario is that the `default` value will often appear in OpenAPI documentation generators.

The `x-template` value must be a string that will have [parameter replacement](#parameter-replacement) occur on it. Parameters in the string may use handlebars, double handlebars, or colons depending on how the Enforcer instance has been [configured](#optionspopulatereplacement).

The `x-variable` will perform value substitution only.

If a conflict arises between the provided value, `default`, `x-template`, or `x-variable` then the following priority is observed:

1. The provided value
2. `x-variable`
3. `x-template`
4. `default`

```js
const Enforcer = require('openapi-enforcer');
const enforcer = new Enforcer('3.0.0');

const schema = {
    type: 'object',
    properties: {
        firstName: {
            type: 'string',
            'x-variable': 'firstName'
        },
        lastName: {
            type: 'string',
            'x-variable': 'lastName'
        },
        fullName: {
            type: 'string',
            'x-template': '{firstName} {lastName}'
        },
        profileUrl: {
            type: 'string',
            default: 'https://your-domain.com/users/{id}'
        }
    }
};

const params = {
    id: 12345,
    firstName: 'Jan',
    lastName: 'Smith'
}

const value = enforcer.populate(schema, params);
// value ==> {
//   firstName: 'Jan',
//   lastName: 'Smith',
//   fullName: 'Jan Smith',
//   profileUrl: 'https://your-domain.com/users/12345'
// }
```

## Enforcer.prototype.serialize

Serialize an value according to the schema. This works for primitives, arrays, and objects. Arrays and objects will be traversed and their values also formatted recursively.

`Enforcer.prototype.serialize ( schema, value )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| schema | The schema to serialize to. | `object` |
| value | The value to format. | Any |

Returns: The serialized value.

Can serialize:

- arrays and objects recursively
- binary from boolean, number, string, or Buffer
- boolean from any value
- byte from boolean, number, string, or Buffer
- date from Date, string, or number
- date-time from Date, string, or number
- integer from anything that !isNaN(value)
- number from anything that !isNaN(value)
- string from string, number, boolean, object, or Date

```js
const Enforcer = require('openapi-enforcer');
const enforcer = new Enforcer('3.0.0');

const schema = {
    type: 'object',
    properties: {
        time: {
            type: 'string',
            format: 'date-time'
        },
        public: {
            type: 'boolean'
        },
        seatsAvailable: {
            type: 'integer'
        }
    }
};

const value = enforcer.serialize(schema, {
    time: new Date(2000, 0, 1, 11), // formatted to ISO Date
    public: 1,                      // formatted to true
    seatsAvailable: 23.7            // formatted to integer
});
// value ==> {
//   startTime: '2000-01-01T11:00:00.000Z',
//   public: true,
//   seatsAvailable: 24
// }
```

### Parameter Replacement

Parameter replacement is when part of a string is populated with parameters. This applies to a schema's `x-template` value and potentially `default` value. There are three types of replacement:

1. handlebar (default)

    ```js
    const Enforcer = require('openapi-enforcer');
    const options = {
        populate: { replacement: 'handlebar' }
    };
    const enforcer = new Enforcer('3.0.0', options);
    
    const schema = {
        type: 'string',
        'x-template': '{name} is {age} years old'
    };
    
    const value = enforcer.populate(schema, { name: 'Bob', age: 25 });
    // value ===> 'Bob is 25 years old
    ```

2. doubleHandlebar

    ```js
    const Enforcer = require('openapi-enforcer');
    const options = {
        populate: { replacement: 'doubleHandlebar' }
    };
    const enforcer = new Enforcer('3.0.0', options);
    
    const schema = {
        type: 'string',
        'x-template': '{{name}} is {{age}} years old'
    };
    
    const value = enforcer.populate(schema, { name: 'Bob', age: 25 });
    // value ===> 'Bob is 25 years old
    ```

3. colon

    ```js
    const Enforcer = require('openapi-enforcer');
    const options = {
        populate: { replacement: 'colon' }
    };
    const enforcer = new Enforcer('3.0.0', options);
    
    const schema = {
        type: 'string',
        'x-template': ':name is :age years old'
    };
    
    const value = enforcer.populate(schema, { name: 'Bob', age: 25 });
    // value ===> 'Bob is 25 years old
    ```

## Enforcer.prototype.schema

## Enforcer.prototype.validate

Validate that the value adheres to the schema or throw an `Error`. This function calls [`enforcer.errors`](#enforcererrors--schema-value-) and if any errors occur then it packages them into a single `Error` instance and throws the `Error`.

`Enforcer.prototype.validate ( schema, value )`

| Parameter | Description | Type |
| --------- | ----------- | ---- |
| schema | The schema to build from | `object` |
| params | A map of keys to values. These values are used to help build the final value | `object` |
| value | An initial value to start with. | Any |

Returns: Nothing.

```js
const OpenApiEnforcer = require('../index');

const enforcer = new OpenApiEnforcer({ openapi: '3.0.0' });

const schema = {
    type: 'object',
    additionalProperties: false,
    properties: {
        names: {
            type: 'array',
            items: {
                type: 'string',
                minLength: 1
            }
        },
        date: {
            type: 'string',
            format: 'date-time',
            maximum: '2000-01-01T00:00:00.000Z'
        }
    }
};

enforcer.validate(schema, {
    names: [ 'Bob', 'Jan', '' ],
    date: '2010-01-01T00:00:00.000Z',
    num: 8
});
// Error: One or more errors found during schema validation: 
//   /names/2: String length below minimum length of 1 with length of 0: ''
//   /date: Expected date-time to be less than or equal to 2000-01-01T00:00:00.000Z. Received: 2010-01-01T00:00:00.000Z
//   /num: Property not allowed
//     at ...
```

## Enforcer.format

Various functions that take a single parameter as input and convert it to a format that is ready for sending as an HTTP response. For example, the `binary` format function will convert a value to a binary string; the date format function will convert a value to a date string.

Note: this static method should not be confused with the [Enforcer.prototype.format](#enforcerprototypeformat) function that formats a value to a provided schema.

| Function | Input |
| -------- | ------|
| binary | A `Boolean`, `Number`, `String`, or `Buffer`. |
| boolean | Any value. |
| byte | A `Boolean`, `Number`, `String`, or `Buffer`. |
| date | A `Date` object, `String` in ISO date or date-time format, or a `Number`. |
| dateTime | A `Date` object, `String` in ISO date or date-time format, or a `Number`. |
| integer | Any value not isNaN. |
| number | Any value not isNaN. |
| string | A `String`, `Number`, `Boolean`, `Object`, of `Date`. |

**Example**

```js
const Enforcer = require('openapi-enforcer');

Enforcer.format.binary(true);       // => "00000001"
Enforcer.format.date(0);            // => "1970-01-01"
Enforcer.format.integer(5.67);      // => 6
```

## Enforcer.is

Various functions that take a single parameter as input and detect if the value fits to a specific type. Some functions have an optional second parameter that can be used to increase the strictness of the check. Returns `true` if of the correct type, otherwise `false`.

| Function | Input | Strict Option |
| -------- | ------| ------------- |
| binary | A binary `String` made up of zeros and ones with a length divisible by 8. | No
| boolean | `"true"`, `"false"`, or a `Boolean`. | Yes - must be a `Boolean` |
| byte | A base64 encoded `String`. | No |
| date | A `Date` object at start of UTC day or `String` in ISO date format. | Yes - must be a `Date` |
| dateTime | A `Date` object or `String` in ISO date or date-time format. | Yes - must be a `Date` |
| integer | An integer `String` or an integer `Number`. | Yes - must be a `Number` |
| number | A `String` or `Number`. | Yes - must be a `Number` |

**Example**

```js
const Enforcer = require('openapi-enforcer');

Enforcer.is.integer("123");         // => true
Enforcer.is.integer(123);           // => true
Enforcer.is.integer(1.23);          // => false
Enforcer.is.integer("123", true);   // => false
```

## Enforcer.parse

Various functions that take a single parameter as input and return the parsed value. Some functions have an optional second parameter that can be used to force parse and allow for more leniency in input.

| Function | Input | Force Option | Return Value |
| -------- | ----- | ------------ | ------------ |
| binary | A binary `String` made up of zeros and ones with a length divisible by 8. | No | `Buffer` |
| boolean | `"true"`, `"false"`, or a `Boolean`. | Yes - any value accepted | `Boolean` |
| byte | A base64 encoded `String`. | No | `Buffer` |
| date | A `Date` object or `String` in ISO date format. | Yes - `Number` or ISO date-time string | `Date` |
| dateTime | A `Date` object or `String` in ISO date-time format. | Yes - `Number` or ISO date string | `Date` |
| integer | An integer `String` or an integer `Number`. | Yes - anything not isNaN | `Number` |
| number | A `String` or `Number`. | Yes - anything not isNaN | `Number` |

**Example**

```js
const Enforcer = require('openapi-enforcer');

Enforcer.parse.integer("123");          // => 123 (as a number)
Enforcer.parse.integer("5.67");         // => throws Error
Enforcer.parse.integer("5.67", true);   // => 6 (as a number)
```