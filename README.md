# FormStorage — Form Serialization and Storage in localStorage

FormStorage is a small JavaScript library for serializing and storing web forms in localStorage, distributed under the MIT license. It intended to save user input and prevent data loss when user suddenly closes window, navigates away from the page, etc when filling out large forms or writing posts and comments to online communities and forums.

The library comprises two classes: `FormSerializer` to handles serialization and deserialization of form into JavaScript object, and `FormStorage` which loads and saves form data from/to localStorage, as and clears the data after successful form submission.

## Serialization with the FormSerializer Class

The FormSerializer class provides two static methods:
* `serialize(form_element)` — accepts an HTMLFormElement object and returns an object containing form data in the format { 'field1': 'value1', 'field2': 'value2', 'field3[]': ['array','of','values'], … }
* `unserialize(form_element, data)` — accepts a form (HTMLFormElement) and populates it with values from the `data` object.

Example:
```javascript
// serialize the contents of the post_form into the variable data
const data = FormSerializer.serialize(document.getElementById('post_form'));
// restore this data into the form my_form2
FormSerializer.unserialize(document.getElementById('my_form2'), data);
```

### Form Serialization Details
* Only fields with `name` attribute are serialized.
* Fields with `autocomplete="off"` are ignored.
* Due to security reasons values of `password` and `file` fields are not saved.
* `input` elements (including checkboxes and radio buttons), `textarea`, and `select` with single or multiple selection are supported.
* FormSerializer handles properly multiple `input` elements with the same name (only if the name contains `[]`).
* Any data attributes, `min` and `max` attributes for range elements, or button values are NOT serialized!
* When loading form data, field with missing key remains unchanged.
* Keys with not matching field found are simply ignored.

## Form Handling with FormStorage

To enable autosave for a form, pass form DOM element and `options` object to FormStorage constructor.
The `form_element` parameter contains the DOM element of the form to save. The `options` object may have the following fields:
* `storage_key` — the key name in localStorage where the form content will be saved. Required.
* `skip_autoloading` —  if `true`, don't load data from localStorage in constructorm, you need to call `load` method explicitly. Default is `false`.
* `headers` — additional HTTP headers to send when submitting form with XMLHttpRequest. Default is none.
* `wait_input` — when `true`, saves form data only after first user input in any field. It is useful to avoid cluttering localStorage with empty form data. Default is `false`.
* `save_on_exit` — if `true`, attach saving handler to `pagehide` event (triggered when the user leaves the page). Default is `true`.
* `autosave_time` — interval for autosave, in seconds. Any value other than positive integer disables timer-based autosave. Default value is 10.

To manually save or load data, you can explicitly call the `load` and `save` methods on the created FormStorage object (no parameters required).

Simple example:
```javascript
const form_storage = new FormStorage(document.getElementById('post_form'), { 'storage_key': 'ExampleFormData' });
```
More advanced example with all settings defined:
```javascript
const form_storage = new FormStorage(document.getElementById('my_form2'),
{ // options parameter
 'storage_key': 'SecondFormData', // key name in localStorage
 'skip_autoloading': true, // do not load the form automatically; we'll do it explicitly later
 'headers': { 'X-Sender': 'FormStorage' }, // add a header so the server can recognize that the form was submitted via XMLHttpRequest
 'wait_input': true, // save the form only after the user has made an input
 'save_on_exit': true,
 'autosave_time': 5, // save the form every 5 seconds instead of 10
}
);
// some conditional check can be placed here to decide whether to load the saved form
form_storage.load(); // explicit form loading
```

## Form Submission, Hooks, and Working with WYSIWYG Editors

FormStorage set an `onsubmit` event handler, which cancels the default browser form processing and sends the data via an AJAX request with the headers from `options.headers`. If the server returns statuses 200, 204, or 206, an automatic redirect to the final URL occurs. For status 201, browser will be redirected to the URL from the `Location` header. If the redirection address differs from the current URL only by the hash part, page will be realoded. In all other cases, the `onSubmitError` error handler is called.
To change this behavior, replace the `onSubmitResult` method with your own (it receives an XMLHttpResult object as a parameter).

Tip: Add a special header to FormStorage requests so the server can distinguish them from regular form submissions. For such requests, it's better to return status 201 with a Location header to avoid double-loading of the result page.

If you need to customize form serialization process, use following hooks:
* **`onBeforeSave`** — called before form serialization starts. No parameters.
* **`onSave`** — called after form serialization but before saving to localStorage. Receives an object with fields to save in the format { "field1": "value", "field2[]": ["array","of","values"] }; you can change the object contents if needed.
* **`onLoad`** — called after parsing data from localStorage before populating the form. Receives an object in the same format as `onSave`.
* **`onSubmitError`** — called when a form submission error occurs (server responds with status other than 200, 201, 204, 206).
* **`onDecodeError`** — called when there's an error parsing JSON data loaded from localStorage.

By default, all hooks are empty (do nothing) except for `onSubmitError`, which logs the error text to the console. You can replace them with your custom code.

Many WYSIWYG editors require to call some method to save content to form before submission or to load text into the editor when the page loads. You can use `onBeforeSave` and `onLoad` hooks for that.
Example hooks for the Quill editor:

```javascript
const quill = new Quill('.quill_full_editor', {
modules: {
  toolbar: "#quill-toolbar",
},
  theme: 'snow',
  placeholder: 'Your new note goes here…',
});

const postform = document.getElementById('postform');
const fstor = new FormStorage(postform, 'FormStorage_draft', { 'Accept': 'application/json' });

fstor.onLoad = function(data) { // override the onLoad hook
  if ('post[text]' in data) quill.root.innerHTML = data['post[text]'];
}

fstor.onBeforeSave = function() { // override the onBeforeSave hook
  // Save the editor's HTML content to the corresponding form field
  document.querySelector('[name="post[text]"]').value = quill.getSemanticHTML();
}
```
