# Motivation & Context

We allow our users to customize their page layouts via Form Templates. Form Templates provide a list of fields and an associated section they should be placed in and can be customized via an editor we provide.

These templates are a core feature of our app and a large portion of pages make use of this functionality. They have a significant impact on the user’s ability to do their job so its is important that they provide a great user experience and are easy to implement.

Form Templates are rendered on the client via two different components:

### 1. InlineComponentForm

The InlineComponentForm renders legacy components to enable editing the form. It tries to dry up its code by using derived state to determine how each field should be rendered. It does this by looking at the model definition.

This is very frequently not sufficient and overrides are added globally to allow overwriting the incorrect configuration. These overrides live in the ux.template service. Overrides are also provided via the FromTemplate model that comes from the server. The server can specify the exact component name that should be rendered for example.

The InlineComponentForm uses the library service to lookup all its data. This means that it expects all options in selects to be pre-fetched.

It relies on the handlebars template for the actual layout of the field sections.

### 2. FormTemplateRender

This component is backed by Luna components. It shifts the rendering mechanism to be similar to Datatable V2 in that is requires the user to configure each field explicitly. It does not rely on derived state and ignores the FromTemplate model overrides that comes from the server.

It relies on the handlebars template to customize what is actually rendered for each field in addition to the section layout itself. Similar to data tables, it has a default render for simple fields, or you may override a specific field and customize how it is displayed. It currently does not make customization of the edit experience simple, and it can lead to poor UX in complex custom scenarios.

FormTemplateRenderer still relies on the library service for its data so all data must be prefetched.

Both of these components have shortcomings. The goal of this design document is to iterate on what FormTemplateRender did and clean up the experience while adding additional capabilities revealed during use:

- Lazy loading of data
- Separation of field customization and layout
- First class edit state customization
- Separation of business logic and rendering logic
- Improved default state handling to dry up each individual invocation
- Supporting traditional forms that has a submit button

## Design & Implementation

Most basic invocation. It takes a template argument which is an array of field definitions for what to display.

```
<Luna::FormRenderer
  @template={{this.template}}
  @model={{@someModel}}
/>
```

The FormRenderer is dumb. It renders what it is being told to render and does not take into consider permissions, form templates, etc. To that end, you will probable want to build the template you pass it and not define it manually:

```js
@cached
get template() {
  const formTemplate = this.args.formTemplate;
  return buildTemplate(formTemplate.left, this.fieldDefinitions);
}
```

Two things to notice here:

1. You pass a section of fields to the buildTemplate method. If you need to render more than one section you will be calling the buildTemplate method multiple times.
2. The fieldDefinitions contain all the configuration for all the possible fields.

This example assumes there is a form template involved. That is not strictly necessary. You can pass in any array of field definitions. The buildTemplate method is intended to simplify the application of applying permissions, configuring each field, etc. The output looks like this:

```js
[
  new TextField("name", { multiline: false }),
  new TextField("description", { multiline: true, linkify: true }),
  new UserField("reviewerUser", { filters: { role: "reviewer" } }),
];
```

Template

The template is an array of field definitions describing what the component should render. A Field is a class that extends from the base Field class. It is straightforward to crease your own custom field by simply extending from the base class. Default implementations will be provided for all the normal field types.

```js
// UserField for example implements a SelectField, but overrides the
// value that is displayed, and implements a custom call to fetch
// the list of users that may be selected
class UserField extend SelectField {
	displayComponent: UserFieldComponent;

	get displayValue() {
		return this.value?.fullName;
	}

	async options(search) {
		return this.store.query('user', {
			filter: { $include: search },
			order: { name: 'ASC' },
			limit: 50
		});
	}
}

// SelectField ensures a select component is used for the edit state
class SelectField extends Field {
	displayComponent: SelectFieldDisplayComponent;
	editComponent: SelectFieldEditComponent;

	get displayValue() {
		const name = get(this.model, this.key)?.name;
		return <template>
				<a href={{href-to "some.path"}}>
					{{name}}
				</a>
			</template>
		}
	}

	// Base class that implements most of the logic around form template data
	class Field {
		constructor(key, model, formTemplate, config) {
			this.key = key;
			this.model = model;
			this.formTemplate = formTemplate;
			this.config = config;

			this.intl = getOwner(model).lookup('service:intl');
			this.store = getOwner(model).lookup('service:store');
			this.permissionManager = getOwner(model).lookup('service:permission-manager');
	}

	@cached
	get formTemplateFieldConfig() {
		return this.formTemplate.find((item) => item.key === this.key);
	}

	get label() {
		return formTemplateFieldConfig.label || this.config?.label;
	}

	get tooltip() {
		return formTemplateFieldConfig.tooltip || this.config?.tooltip;
	}

	get value() {
		return get(this.model, this.key);
	}

	get displayValue() {
		return this.value;
	}

	get placement() {
		return this.config?.placement || this.config?.placement;
	}

	@cached
	get isVisible() {
		if (!formTemplateFieldConfig) {
			return false;
		}

		if (this.formTemplateFieldConfig.requiredChecks && !this.permissionManager.check(this.formTemplateFieldConfig.requiredChecks)) {
			return false;
		}

		if (this.config?.requiredChecks && !this.permissionManager.check(this.config.requiredChecks)) {
			return false;
		}

		return true;
	}
}
```
