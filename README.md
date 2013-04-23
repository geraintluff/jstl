# jstpl - a JavaScript templating library

`jstpl` is a library for templating for JavaScript, *using* JavaScript.

Rather than trying to design a new language, `jstpl` allows you to write JavaScript code for the control logic of your templates.

`jstpl` takes in a template, and produces a function.  This function **includes JavaScript code from the template**, so it is basically equivalent to `eval()` - **DO NOT RUN TEMPLATES YOU DO NOT TRUST**.

Example code:
```html
<div class="this-user">
    <div class="name"><?fullname?></div>
    <div class="security-status">
        <?js if (window.location.substring(0, 5) == "https") { ?>
        <div class="security-status-encrypted">Encrypted</div>
        <?js } else { ?>
        <div class="security-status-unencrypted">Unencrypted</div>
        <?js } ?>
    </div>
</div>
```

## *Another* templating library?

Yeah, I know there are already quite a few.  But they generally suffer from the following problems:

* **New syntax** - the more complex ones define their own syntax for `if`/`foreach`/etc.
* **Not customisable** - they give you a set of pre-defined output filters (HTML entities, URI encoding, etc).

`jstpl` tries to avoid these, by piggybacking on the JavaScript language while providing a simple base for extending behaviour.

## Writing templates

There are two different ways to use the special tags in a template.

### 1) No space: `<?variable?>` or `<?jsvariable?>`

If there is no space following the `<?` (or `<?js`), the block is substituted with a value.  By default, this is taken as a property of the first function argument (HTML-escaped).

Example template:
```html
<div>
    <?name?> likes <?food?>.
</div>
```

Calling the function:
```javascript
var obj = {name: "Everybody", food: "chocolate"};
var html = myTemplate(obj);
```

Result:
```html
<div>
    Everybody likes chocolate.
</div>
```

*Note that if your variable name begins with "js", you will need to use the long form, e.g. `<?jsjsvar?>`*

### 2) With space: `<? ... ?>` or `<?js ... ?>`

If there is whitespace following the opening `<?` or `<?js`, the block is run as JS code.

The special function `echo()` is defined, which contribues to the output of the function.

Example template:
```html
<div>
    Current URI: <?js
        echo(window.location.toString());
    ?>
</div>
```

Result:
```html
<div>
    Current URI: http://example.com/
</div>
```

## Loading templates

Templates can be loaded either as strings (`jstpl.create(str)`) or they can be loaded from C-style comments in your JS file, assuming that file is accessible via AJAX.

The syntax for loading from comments is to start your comment `/* Template: <identifier>` followed by a newline.  To load all templates in a file, call `jstpl.loadTemplates()` from that file.

Example:
```javascript
template = jstpl.create('<div><?title?></div>').compile();

/* Template: example-template
<div>
    <?title?>
</div>
*/
template2 = jstpl.getTemplate('example-template').compile();
```

## Customising

Behaviour can be modified either by passing arguments into `template.compile()`, or by changing properties in the `jstpl` object.

### The "no spaces" behaviour

You can completely customise the behaviour of the "no spaces" form.

The way you do this is by passing in a function to `t.compile()` of the following form:
```javascript
t.compile(function (variableName) {
    return function (arg0, arg1, ...) {
        ...
    }
});
```

You might need to look at that twice - you supply a compile-time function which returns a function that is called at render time.

For example, a simple example might be:
```javascript
var t = jstpl.create('Name: <?name?>');
var f = t.compile(function (variableName) {
    return function (value) {
        return value[variableName].toString();
    }
});

var value = {name: "Mary"};
var result = f(value); // returns: "Name: Mary"
```

The default function (that `t.compile()` uses if you don't supply your own) is available at `jstpl.defaultFunction`.  It looks very much like the above, except it HTML entity-encodes the output.

You can change this default by altering `jstpl.defaultFunction`.

### "Header code"

You can also provide some JavaScript code to sit at the top of the compiled template.  It's completely equivalent to adding the same code to the top of the template.

```javascript
var f = t.compile(null, "var value = arguments[0];");
```

A good use for this might be to alias function arguments (as in the example) - that makes the `value` variable available for use within your templates, so you don't have to pepper your code with `arguments[N]`.

The default "header code" is actually the one from above example - however, you can change this by altering `jstpl.defaultHeaderCode`.

## Wait - how does it get the templates from JavaScript comments?

The secret is that when `jstpl.getTemplate()` is called, it calls through to `jstpl.loadTemplates()`.  This function figures out what source file is running (it should be the last `<script>` in the document), fetches it via AJAX and inspects it for comments.

If for some reason you aren't calling `jstpl.getTemplate()` when the script loads (I can't think why you would), then you should call `jstpl.loadTemplates()` manually.