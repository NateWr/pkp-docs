# Introduction to Schema

This document describes proposed changes to define entities in OJS/OMP using an extended form of `json-schema`. These machine-readable descriptions can then be used to sanitize, validate and read/write to the database.

It also briefly introduces new form components built in Vue.js and describes how they can be instantiated and extended alongside an entity schema.

## Table of Contents

- [Getting Started](#getting-started)
- [json-schema](#json-schema)
- [Validation](#validation)
- [SchemaDAO](#schemadao)
- [Bringing together API endpoints, Service classes, and Schema](#bringing-together-api-endpoints-service-classes-and-schema)
- [Forms](#forms)
- [Extending Schemas and Forms](#extending-schemas-and-forms)
- [Known Issues](#known-issues)

## Getting Started

You will need to check out the following PRs:

- ojs: https://github.com/pkp/ojs/pull/2067
- pkp-lib: https://github.com/pkp/pkp-lib/pull/3931
- ui-library: https://github.com/pkp/ui-library/pull/20

You will need to install node dependencies and build the JS:

```
cd <ojs-directory>
npm install
npm run dev
```

I don't think you need to update composer dependencies but maybe do it just in case.

Also, you need to drop the `setting_type` column from the `journal_settings` table. For a less destructive option, you can get by just altering the column to allow `NULL` values.

I encourage you to look at the UI Library to see an example form that includes all of the existing form fields. To view that:

```
cd <ojs-directory>/lib/ui-library
npm install
npm run dev
```

## json-schema

Entities are defined using [json-schema](http://json-schema.org/), which mirrors the format we have been using for our API documentation. (We've been using YAML, but the structure of it mirrors json-schema and the tooling can accept json-schema). Here are some [quick examples](http://json-schema.org/learn/getting-started-step-by-step.html).

Schemas are stored in a new top-level directory, `schemas`. When the application finds a schema with the same name in `<root>/schemas` and `<root>/lib/pkp/schemas`, the schemas are merged.

So far, a context is the only entity described by a schema:

- [base context schema in pkp-lib](https://github.com/NateWr/pkp-lib/blob/i3594_forms/schemas/context.json)
- [additional properties defined in context schema in ojs](https://github.com/NateWr/pkp-lib/blob/i3594_forms/schemas/context.json)

### SchemaService

A new [SchemaService](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPSchemaService.inc.php) class has been created to load and merge schemas, validate properties against a schema, and more.

A schema can be loaded using the service class and the schema constants:

```
$schemaService = ServicesContainer::instance()->get('schema');
$schema = $schemaService->get(SCHEMA_CONTEXT);
```

### Extending json-schema

To serve our needs, we have introduced a few properties and practices that are not part of the official `json-schema` spec:

#### `multilingual`
Assigned to multilingual values. This property changes how a value is validated and sanitized. For example, consider the following property definition:

```
"about": {
	"type": "string",
	"multilingual": true
},
```

When validating and sanitizing this property, it will be expected to be an object. Each key in the object must match an active locale, and the value of each key will be validated against the `type` (string). So the following value would be expected in PHP:

```
[
	'en_US': 'Some words about this journal.',
	'fr_CA': 'Quelques mots sur ce journal.'
]
```

**Data described as an object in `json-schema` is expected to be an assoc array in PHP, rather than a proper PHP object.**

The `multilingual` property can only be used on top-level properties. If a property is an object or array, the whole property must be multilingual, not an individual sub-property or a single item in the array.

#### `apiOnly`
When this key is assigned to a property, the property is only expected to appear in API responses. It will not be considered an accepted property when validating user input.

#### `apiSummary`
When this key is assigned to a property, the property will appear in summary views of the entity that are provided in the API. For example, when many contexts are listed in `/api/v1/contexts`.

#### `defaultLocaleKey`
This key is assigned to a property when a default value must be localised.

#### `patternErrorLocaleKey`
This key is assigned when a property is validated according to a regex pattern. See [Validation](#validation) below.

#### Date and time formats
Instead of the standard `date` and `date-time` formats, we use `date-iso` (`YYYY-MM-DD`) and `date-time-iso` (`YYYY-MM-DD HH:MM:SS`) to more strictly match the way we handle dates and times in our apps.

## Validation

`json-schema` comes with a number of standard [validation rules](http://json-schema.org/latest/json-schema-validation.html). We support a [limited sub-set of these rules](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPSchemaService.inc.php#L24-L48).

We support all of the number and string rules. And I plan to implement the generic `type`, `enum` and `const` keywords ([described here](http://json-schema.org/latest/json-schema-validation.html#rfc.section.6.1.3)).

Looking at the array and object validation rules, I felt that they had limited use-cases and required writing some pretty complicated code. So I felt we could implement them on an as-needed basis.

I did find two PHP libraries for json-schema validation, but chose not to use them. [opis/json-schema](https://github.com/opis/json-schema) requires PHP 7. [swaggest/php-json-schema](https://github.com/swaggest/php-json-schema) looked promising, but I found it's API kind of impenetrable and there were a few problems that I felt would take a lot of work to get around for our particular usage. The main issue was that it seemed very difficult to tie validation errors with user-facing error messages. In the end, the simple validation rules we needed were the easiest part of this process so I felt ok rolling our own solution.

### Error messaging
Most of the validation rules allow for us to provide sensible error messages in a generic way. Consider the following property schema:

```
"licenseUrl": {
	"type": "string",
	"format": "uri"
},
```

When this validation fails, `SchemaService::validate()` knows to return the following error message:

```
Please provide a valid URL, such as https://example.com.
```

In some cases, we need to pass a more explicit error message. Consider the following property schema, which validates against a regex:

```
"path": {
	"type": "string",
	"pattern": "^[a-z0-9]+([\\-_][a-z0-9]+)*$",
},
```

In this case, we need a user-facing description of what we expect the value to look like. To provide this, we use a non-standard key, `patternErrorLocaleKey`, to provide an error message:

```
"path": {
	"type": "string",
	"pattern": "^[a-z0-9]+([\\-_][a-z0-9]+)*$",
	"patternErrorLocaleKey": "admin.contexts.form.pathAlphaNumeric",
},
```

However, in some cases the schema can not fully describe the validation we must perform against an entity. For example, a context `path` be must unique and we require a `name`, but only in the primary locale. I chose not to define a custom validation property for every potential case like this. The SchemaService doesn't know where to look for duplicates anyway. Instead, we perform additional validation actions in the entity's service class, after the schema validation has occurred. You can see an example of this in `PKPContextService::validate()`:

https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L200-L254

## SchemaDAO

With the schema, it becomes possible to automate the most common read/write actions in the database. To explore this approach, I've created a new [SchemaDAO](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/db/SchemaDAO.inc.php) class and `ContextDAO` now extends from that.

A `SchemaDAO` expects some configuration properties which tell it what schema to get, what tables it interacts with, etc. Then it can perform the `insertObject`, `updateObject`, `deleteObject`, and `_fromRow` methods without any further work.

Now that we use the new `QueryBuilder` approach for complex selects, it may be possible to implement all or most of our DAOs without any further code in the DAO.

Based on conversations with Dmitris, we may also want to define the relationship betweeen a schema entity and its databases, as well as associated entities, in the schema itself. If we do that, we may be able to write a single DAO class that reads the configuration properties directly from the schema. I'm definitely interested in where you think we might go further with this. It's not something I considered in the initial work but makes a lot of sense.

## Bringing together API endpoints, Service classes, and Schema

The API's [PKPContextHandler](https://github.com/NateWr/pkp-lib/blob/i3594_forms/api/v1/contexts/PKPContextHandler.inc.php) is the best demonstration of how the schemas, services and DAOs are meant to work together. The route handler for [addContext](https://github.com/NateWr/pkp-lib/blob/i3594_forms/api/v1/contexts/PKPContextHandler.inc.php#L181-L218) is a good demonstration of how it calls on ContextService to do the work. But the ContextService itself relies on SchemaService and SchemaDAO objects to do most of the work.

- `PKPContextService::validate()` relies on SchemaService to do [most of the work](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L219), only adding in a few validation checks that the schema can't define.
- `PKPContextService::addContext()` has some glue to ensure [default locale keys have the required params](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L279-L291), but it relies on the SchemaService to [set defaults](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L296-L304).

A short-hand map of API calls shows the parts of the process which overlap:

```
GET:    Request -> Auth -> Get -> Return
ADD:    Request -> Auth -> Validate -> Sanitize -> Add -> Get -> Return
EDIT:   Request -> Auth -> Validate -> Get -> Merge -> Sanitize -> Edit -> Get -> Return
DELETE: Request -> Auth -> Validate -> Delete -> Return
```

## Forms

I won't describe forms fully in this doc. But they communicate with the API, so that they take advantage of the validation available in the schema. For example, when we set up the Settings > Journal > Contact form we tell it to [submit to the `/contexts` api](https://github.com/NateWr/pkp-lib/blob/i3594_forms/controllers/tab/settings/ManagerSettingsTabHandler.inc.php#L128).

We can [pass a message to display when the form is successfully submitted](https://github.com/NateWr/pkp-lib/blob/i3594_forms/controllers/tab/settings/ManagerSettingsTabHandler.inc.php#L129).

When submitted to the API, if there are validation errors, they will be received back in a JSON object like this:

```
{
	"contactEmail": "Please provide a valid email address."
}
```

The Vue.js form handler knows how to display this error message in the forms by matching the property name to the field.

There's lots of work to do still here (see [Known Issues](#known-issues)). But that's a quick look at how the schemas are reused on the form side.

## Extending Schemas and Forms

Schemas can be extended by plugins to add, edit or remove the properties of an entity (see [hook](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPSchemaService.inc.php#L89)).

I built a small [example plugin](https://github.com/NateWr/institutionalHome) to demonstrate how data can be attached to entities (as well as the input forms).

Once the schema is modified, any additional actions are hooked to methods in the service class. These include [building a JSON representation of an object](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L164), [validation](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L251), [adding](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L332), [editing](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L356), and [deleting](https://github.com/NateWr/pkp-lib/blob/i3594_forms/classes/services/PKPContextService.inc.php#L407).

## Known Issues
I'm aware of the following. There are probably lots of other issues to sort out.

- Currently the workflow pages are 404'ing! I just haven't looked into it yet.
- When a new form is embedded in an old tab, it causes an error and you can't browse between tabs. I will solve that OR just move to Vue.js tabs.
- When adding a context, the form/submission languages don't seem to get setup properly.
- Journal settings wizard is broken due to the new masthead form.
- SchemaDAO: we discussed extracting only data for active locales. I found that a bit challenging given how things are passed through the DAOResultFactory so for the moment we still pull out data from any locales.
- The form components are not yet properly documented in the UI Library.
- I have not yet looked at using the json-schema in our API documentation. However, the Swagger documentation tool we use accepts json objects in this form, so it should be fairly easy to compile the necessary doc file.
- You may have noticed that with the schema we can get away with things like `$context->_data = $params`, `$context->getData('name')`, etc. I wonder if it would be a good or bad idea to consider this approach the standard and to get rid of the individual method calls like `$context->getName()`?
- It should be possible for us to add json syntax validation to our tests using an npm tool. In the meantime, https://jsonlint.com/ works great.
