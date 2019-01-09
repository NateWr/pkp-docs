# Introduction to Schema

This document describes a new way to define entities in OJS/OMP using an extended form of `json-schema`. These machine-readable descriptions can then be used to sanitize, validate and read/write to the database.

It also briefly introduces new form components built in Vue.js and describes how they can be instantiated and extended alongside an entity schema.

## Table of Contents

- [Getting Started](#getting-started)
- [json-schema](#json-schema)
- [Validation](#validation)
- [SchemaDAO](#schemadao)
- [Bringing together API endpoints, Service classes, and Schema](#bringing-together-api-endpoints-service-classes-and-schema)
- [Forms](#forms)
- [SettingsContainer for forms and tabs](#user-content-settingscontainer-for-forms-and-tabs)
- [Extending Schemas and Forms](#extending-schemas-and-forms)

## Getting Started

If you haven't yet, update the submodules:

```
cd <ojs-directory>
git submodule update --init --recursive
```

Update composer dependencies in `lib/pkp`:

```
cd lib/pkp
composer update
```

Install new node dependencies and build the JS:

```
cd <ojs-directory>
npm install
npm run build
```

Run the upgrade tool (backup your DB if you need to work on 3.1.2):

```
php tools/upgrade.php upgrade
```

When you have time, I encourage you to look at the UI Library to see documentation on the forms. To view that:

```
cd <ojs-directory>/lib/ui-library
npm install
npm run dev
```

## json-schema

Entities are defined using [json-schema](http://json-schema.org/). Here are some [quick examples](http://json-schema.org/learn/getting-started-step-by-step.html).

Schemas are stored in a new top-level directory, `schemas`. When the application finds a schema with the same name in `<root>/schemas` and `<root>/lib/pkp/schemas`, the schemas are merged. This is how an application can add properties to a base entity.

Several entities have been defined in a schema to assist the API documentation, but the only entities that are fully running off of schemas for DAO, validation and sanitization are Contexts and Sites:

- Context: [pkp-lib](https://github.com/pkp/pkp-lib/blob/master/schemas/context.json), [ojs](https://github.com/pkp/ojs/blob/master/schemas/context.json)
- Site: [ojs](https://github.com/pkp/ojs/blob/master/schemas/site.json)

### SchemaService

A new [SchemaService](https://github.com/pkp/pkp-lib/blob/master/classes/services/PKPSchemaService.inc.php) class has been created to load and merge schemas, validate properties against a schema, and more.

A schema can be loaded using the service class and the schema constants:

```
$schemaService = Services::get('schema');
$schema = $schemaService->get(SCHEMA_CONTEXT);
```

### API Documentation

Our API documentation has been converted to `json`. A new CLI tool, `tools/buildSwagger.php`, will compile the API documentation using `docs/dev/swagger-source.json` and the entity schema files. This can then be copied to the `pkp-docs` repository to update the documentation.

### Extending json-schema

To serve our needs, we have introduced a few properties and practices that are not part of the official `json-schema` spec:

#### `multilingual`
Assigned to multilingual properties. This property changes how a value is validated and sanitized. For example, consider the following property definition:

```
"about": {
	"type": "string",
	"multilingual": true
},
```

When validating and sanitizing this property, an object will be expected. Each object key must match an active locale, and the value of each key will be validated against the `type` (string, in this case). So the following value would be expected in PHP:

```
[
	'en_US': 'Some words about this journal.',
	'fr_CA': 'Quelques mots sur ce journal.'
]
```

**Data described as an object in `json-schema` is expected to be an assoc array in PHP, rather than a proper PHP object.**

The `multilingual` property can only be used on top-level properties. If a property is an object or array, the whole property must be multilingual, not an individual sub-property or a single item in the array.

#### `readOnly`
The property is only expected to appear in API responses. It will not be considered an accepted property when validating user input.

#### `writeOnly`
The property is only expected to be sent in requests to add/edit an object. It will not be returned when requesting an object.

This property is used for cases where a user may submit information that needs to be processed. An example is the `temporaryFileId` that is used to save a context file to the public directory, but is then discarded.

#### `apiSummary`
The property will appear in summary views of the entity that are provided in the API. For example, when many contexts are listed in `/api/v1/contexts`.

#### `defaultLocaleKey`
The default value for this property must be localised. The value should be a locale string.

#### `validation`
This key describes validation rules that should be applied to the property. We do not support `json-schema`'s standard validation rules. See [Validation](#validation) below.

#### Date and time formats
Instead of `json-schema`'s standard `date` and `date-time` formats, we use `date-iso` (`YYYY-MM-DD`) and `date-time-iso` (`YYYY-MM-DD HH:MM:SS`) to more strictly match the way we handle dates and times in our apps.

## Validation

We use the [Validation library from Laravel 5.5](https://laravel.com/docs/5.5/validation). We support all of the [validation rules](https://laravel.com/docs/5.5/validation#available-validation-rules) except for those which validate against the database. (These require Laravel's entity and database setup.)

Validation rules should be defined in the schema with an array:

```
...
	"itemsPerPage": {
		"type": "integer",
		"default": 25,
		"validation": [
			"nullable",
			"min:1"
		]
	},
...
```

All properties that may be assigned an empty value should have a `nullable` rule. For example, a `PUT` request with the following body will delete the `itemsPerPage` row from the `journal_settings` database:

```
{
	"itemsPerPage": ""
}
```

Type validation will occur automatically based on the `type` property. The following `integer` validation is redundant:

```
...
	"itemsPerPage": {
		"type": "integer",
		"default": 25,
		"validation": [
			"integer",
			"nullable",
			"min:1"
		]
	},
...
```

### Validating an object

Each Service class will have a `validate` method which can validate properties passed to it. The example below shows how a `PUT` request to `/contexts/1` is handled:

```
$params = $this->convertStringsToSchema(SCHEMA_CONTEXT, $slimRequest->getParsedBody());
$params['id'] = (int) $args['contextId'];
$errors = $contextService->validate(VALIDATE_ACTION_EDIT, $params, $allowedLocales, $primaryLocale);
if (!empty($errors)) {
	return $response->withJson($errors, 400);
}
```

The `validate` method draws on `ValidatorFactory`, which can be used to validate data against a rule-set. The example below shows how the schema is used to generate the validation rules:

```
$schemaService = Services::get('schema');
import('lib.pkp.classes.validation.ValidatorFactory');
$validator = \ValidatorFactory::make(
	$props,
	$schemaService->getValidationRules(SCHEMA_CONTEXT, $allowedLocales),
	[
		'path.regex' => __('admin.contexts.form.pathAlphaNumeric'),
		'primaryLocale.regex' => __('validator.localeKey'),
		'supportedFormLocales.regex' => __('validator.localeKey'),
		'supportedLocales.regex' => __('validator.localeKey'),
		'supportedSubmissionLocales.*.regex' => __('validator.localeKey'),
	]
);
```

`ValidatorFactory::make()` is designed to align with [Laravel's `Validator::make()` method](https://laravel.com/docs/5.5/validation#manually-creating-validators). The third argument in the example above matches how Laravel [accepts custom error messages](https://laravel.com/docs/5.5/validation#custom-error-messages).

### Helper methods

In addition to the standard Laravel-style validation rules, `ValidatorFactory` includes helper methods for checking rules around our app's multilingual support. These include:

- `ValidatorFactory::required()` Check for required fields with proper handling of multilingual fields
- `ValidatorFactory::allowedLocales()` Check that no input has been sent from an unsupported locale
- `ValidatorFactory::requirePrimaryLocale()` Pass an array of keys for props that should require the primary locale when data for any other locale exists

### Custom Validation Rules

New global validation rules can be written in `ValidatorFactory::make()`. OJS includes the following:

- `email_or_localhost` (extends Laravel's email validation to accept `localhost`)
- `issn`
- `orcid`
- `currency` (checks if the value is a recognized currency code)

### Entity-specific Validation Rules

In some cases, validation is required that can not be declared in the `json-schema`. These rules should be declared in the Service class's `validate` method using the [after validation hook](https://laravel.com/docs/5.5/validation#after-validation-hook).

For example, two journals can not have the same `path`. To validate a new `path` value, we have to check the database. The example below, which appears in `PKPContextService::validate()`, shows how to check the `path` property:

```
$validator->after(function($validator) use ($action, $props) {
	if (isset($props['path']) && !$validator->errors()->get('path')) {
		$contextDao = Application::getContextDAO();
		$contextWithPath = $contextDao->getByPath($props['path']);
		if ($contextWithPath) {
			if (!($action === VALIDATE_ACTION_EDIT
					&& isset($props['id'])
					&& (int) $contextWithPath->getId() === $props['id'])) {
				$validator->errors()->add('path', __('admin.contexts.form.pathExists'));
			}
		}
	}
});
```

### Applying validation anywhere

Validation can be applied anywhere by invoking `ValidatorFactory::make()`:

```
import('lib.pkp.classes.validation.ValidatorFactory');
$validator = \ValidatorFactory::make(
	['value' => $value],
	['value' => 'issn']
);
if ($validator->fails()) {
	...
}
```

However, this practice should be *discouraged* in most cases. Whenever an object is validated, it should be passed though its Service class's `validate` method to ensure the appropriate hooks are fired.

### Deprecating the `Validator` classes

Some of the old `Validator` classes have been removed. Those that remain are used by a `FormValidator`. They have been refactored to use Laravel's validation library under-the-hood.

## SchemaDAO

With the schema, it becomes possible to handle the most common read/write actions in the database with a parent DAO class. To explore this approach, I've created a new [SchemaDAO](https://github.com/pkp/pkp-lib/blob/master/classes/db/SchemaDAO.inc.php) class. `ContextDAO` extends it.

A `SchemaDAO` expects some configuration properties which tell it what schema to get, what tables it interacts with, etc. It then runs the `insertObject`, `updateObject`, `deleteObject`, and `_fromRow` methods based on the entity definition in the schema.

Now that we use the new `QueryBuilder` approach for complex selects, it may be possible to implement all or most of our DAOs with common code in `SchemaDAO`.

`SiteDAO` could not take advantage of `SchemaDAO` because there is no `site_id`. I felt it was an exception and hope the `SchemaDAO` approach will prove useful in just about every other object we use.

## Bringing together API endpoints, Service classes, and Schema

[PKPContextHandler](https://github.com/pkp/pkp-lib/blob/master/api/v1/contexts/PKPContextHandler.inc.php) is the best demonstration of how the schemas, services and DAOs are meant to work together to handle an API request. [ManagementHandler](https://github.com/pkp/pkp-lib/blob/master/pages/management/ManagementHandler.inc.php) is the best example of a Page request.

## Forms

`FormComponent` is a new class for assembling a form and generating the configuration object that needs to be passed to the Vue component on the frontend. It uses chaining like the `QueryBuilder` to compose forms:

```
$masthead = new FormComponent(
	FORM_MASTHEAD,
	'PUT',
	$contextApiUrl,
	__('manager.setup.masthead.success'),
	$supportedLocales
);
$masthead->addField(new FieldText('name', [
		'label' => __('manager.setup.contextName'),
		'size' => 'large',
		'isRequired' => true,
		'isMultilingual' => true,
		'value' => $context->getData('name'),
	]))
	->addField(new FieldText('acronym', [
		'label' => __('manager.setup.journalInitials'),
		'size' => 'small',
		'isRequired' => true,
		'isMultilingual' => true,
		'value' => $context->getData('acronym'),
	]));
```

Each of the `Field` child classes correspond to a Vue component in the UI Library.

### Built-in Forms

Built-in forms extend the `FormComponent` and encapsulate the setup in the constructor, so that dependencies can be documented clearly and app-specific requirements can be managed with the typical `Class extends PKPClass` method. An example is [MastheadForm](https://github.com/pkp/ojs/blob/master/classes/components/forms/context/MastheadForm.inc.php) and [PKPMastheadForm](https://github.com/pkp/pkp-lib/blob/master/classes/components/forms/context/PKPMastheadForm.inc.php).

### /components

`FormComponent` and its child classes are stored in `/components`, a new directory for UI classes. These classes do not handle requests or modify data. They are only called upon to assemble data that will be passed to Vue components. The `ListHandler` has been changed to `ListPanel` and moved into this directory.

### Groups and Pages

Forms support field groups (`$form->addGroup()`) and pages (`$form->addPage()`). Further documentation to come, but running `$form->getConfig()` will set everything up properly whether or not you've added groups and pages.

### API

Forms always send a `PUT` or `POST` request to the API and expect to receive one of the following responses:

- A `200` response when successful with a JSON object describing the entity that was added or edited.
- A `403` or `404` response when the server refuses the submission with a JSON object describing the error.
- A `400` response when a validation error occurs with one of the fields. In this case, a JSON object will be returned with each invalid field as a key and an array of errors for that field.

Further details in the [API Documentation](https://docs.pkp.sfu.ca/dev/api/ojs/dev).


## Vue.js container component for Forms and Tabs

To get a form onto the page, we pass the form data into a `Container` component, which can manage data for multiple forms.

In the example below, we create our form and prepare our template data:

```
import('lib.pkp.components.forms.context.PKPContactForm');
$contactForm = new PKPContactForm($apiUrl, $locales, $context);
import('components.forms.context.MastheadForm');
$mastheadForm = new MastheadForm($apiUrl, $locales, $context);
$settingsData = [
	'forms' => [
		FORM_CONTACT => $contactForm->getConfig(),
		FORM_MASTHEAD => $mastheadForm->getConfig(),
	],
];
$templateMgr->assign('settingsData', $settingsData);
```

In the example below, we compose our settings page in a `.tpl` file and pass the data into the container:

```
{assign var="uuid" value=""|uniqid|escape}
<div id="settings-context-{$uuid}">
	<tabs>
		<tab id="masthead" name="{translate key="manager.setup.masthead"}">
			<pkp-form
				v-bind="forms.{$smarty.const.FORM_MASTHEAD}"
				@set-fields="setFormFields"
				@set-errors="setFormErrors"
				@set-visible-locales="setFormVisibleLocales"
			/>
		</tab>
		<tab id="contact" name="{translate key="about.contact"}">
			<pkp-form
				v-bind="forms.{$smarty.const.FORM_CONTACT}"
				@set-fields="setFormFields"
				@set-errors="setFormErrors"
				@set-visible-locales="setFormVisibleLocales"
			/>
		</tab>
	</tabs>
</div>
<script type="text/javascript">
	pkp.registry.init('settings-context-{$uuid}', 'Container', {$settingsData|json_encode});
</script>
```

## Extending Schemas and Forms

Schemas can be extended by plugins to add, edit or remove the properties of an entity.

I built a small [example plugin](https://github.com/NateWr/institutionalHome) to demonstrate how data can be attached to entities and then added to a form.

Once the schema is modified, any additional actions are hooked to methods in the service class and DAO so that when the object is fetched, validated, added, edited or deleted, the new property is taken into account.
