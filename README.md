# jstl - JavaScript as a Templating Language

`jstl` is a templating library for JavaScript, *using* JavaScript.

Rather than trying to design a new language, `jstl` allows you to write JavaScript code for the control logic of your templates, while also providing a customisable "short form" for variable substitution.

`jstl` takes in a template, and produces a function.  This function **includes JavaScript code from the template**, so it is basically equivalent to `eval()` - **DO NOT RUN TEMPLATES YOU DO NOT TRUST**.

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

Everywhere `<?js` is used, `<?` or `<%` can be used instead (and `%>` can be used in place of `?>`).

## *Another* templating library?

Yeah, I know there are already quite a few.  But they generally suffer from the following problems:

* **New syntax** - the more complex ones define their own syntax for `if`/`foreach`/etc.
* **Not customisable** - they give you a set of pre-defined output filters (HTML entities, URI encoding, etc).

`jstl` tries to avoid these, by piggybacking on the JavaScript language while providing a simple base for extending behaviour.

## Writing templates

There are two different ways to use the special tags in a template.

### 1) Simple substitution: `<?variable?>`, `<?jsvariable?>` or `<%variable%>`

If there is no space following the `<?` (or `<?js`), the block is substituted with a value.  By default, this is taken as a property of the first function argument (HTML-escaped).

Example template:
```html
<div>
    <%name%> likes <%food%>.
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

*Note that if your variable name begins with "js", you will need to either use the long form (`<?jsjsvar?>`) or the percent form (`<%jsvar%>`)*

### 2) JavaScript code: `<? ... ?>`, `<?js ... ?>` or `<% ... %>`

If there is whitespace following the opening `<?`/`<?js`/`<%`, the block is run as JS code.

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

Templates can be loaded either as strings (`jstl.create(str)`) or they can be loaded from C-style comments in your JS file, assuming that file is accessible via AJAX.

The syntax for loading from comments is to start your comment `/* Template: <identifier>` followed by a newline.  To load all templates in a file, call `jstl.loadTemplates()` from that file.

Example:
```javascript
template = jstl.create('<div><?title?></div>').compile();

/* Template: example-template
<div>
    <?title?>
</div>
*/
template2 = jstl.getTemplate('example-template').compile();
```

## Customising

Behaviour can be modified either by passing arguments into `template.compile()`, or by changing properties in the `jstl` object.

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
var t = jstl.create('Name: <%name%>');
var f = t.compile(function (variableName) {
    return function (value) {
        return value[variableName].toString();
    }
});

var value = {name: "Mary"};
var result = f(value); // returns: "Name: Mary"
```

The default function (that `t.compile()` uses if you don't supply your own) is available at `jstl.defaultFunction`.  It looks very much like the above, except it HTML entity-encodes the output.

You can change this default by altering `jstl.defaultFunction`.

### "Header code"

You can also provide some JavaScript code to sit at the top of the compiled template.  It's completely equivalent to adding the same code to the top of the template.

```javascript
var f = t.compile(null, "var value = arguments[0];");
```

A good use for this might be to alias function arguments (as in the example) - that makes the `value` variable available for use within your templates, so you don't have to pepper your code with `arguments[N]`.

The default "header code" is actually the one from above example - however, you can change this by altering `jstl.defaultHeaderCode`.

## Wait - how does it get the templates from JavaScript comments?

### In-browser

The secret is that when `jstl.getTemplate()` is called, it calls through to `jstl.loadTemplates()`.  This function figures out what source file is running (it should be the last `<script>` in the document), fetches it via AJAX and inspects it for comments.

If for some reason you aren't calling `jstl.getTemplate()` when the script loads (I can't think why you would), then you should call `jstl.loadTemplates()` manually, optionally including the URI of the source file containing templates.

### Using Node

Since the above AJAX trick doesn't work when running server-side, every script containing templates should call `jstl.loadTemplates(__filename)`, so the library can read the file and parse all templates from it.