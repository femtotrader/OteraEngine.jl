# Tutorial

## Installation

OteraEngine can be installed using the Julia package manager. From the Julia REPL, type ] to enter the Pkg REPL mode and run.
```
pkg> add OteraEngine
```

## API

Acutually, this package has only one structure, but these are very powerful because of Metaprogramming function of Julia.

```@docs
Template
@filter
```

Learn about syntax and configuration in the sections below.

## Abstract for Usage
Basic syntax of OteraEngine is very similar to one of Jinja2 of Python. You can use OteraEngine for any extension format which has text.
There are 3 types of blocks:
- `{% ... %}`: Control block
- `{{ ... }}`: Expression block
- `{# ... #}`: Comment block
Control block is used for control flow like `if` or `for`, and Expression block is used for embedding variables. Commend block is just ignored and remove from template.
These block must be familiar with those who have ever used jinja2, but OteraEngine has one more block.

!!! compat "Julia block is no longer supported(since v1.0.0)"
    Julia blocks have been deprecated to simplify the renderer. Now, you have to pass variables processed in caller.

## Variables
As mentioned in previous section, you can embed variables with Expression block. And you can define variables in both templates and julia. Here is an example:
```html
<div>
    {% let name = "Julia" %}
        Hello {{name}}
    {% end %}
</div>
```
You can define variables from julia code like this:
```julia
tmp = Template(...)
tmp(init=Dict(:name=>"Julia"))
```
init is also used for control blocks, and its type is `Dict`. The format is `(variable symbol)=>(value)`.

## Filters
This is very useful function for expression block. You can apply filters for variables like this:
```
{{ value |> escape }}
```
variable name and filter name are separeted by `|>`. Built-in filters are followings:
- `escape` or `e`: escape variables with `Markdown.htmlesc`
- `upper`: convert variables into uppercase
- `lower`: convert variables into lowercase
- `safe`: protect literal html fragments (from autoescape)
- `quote_sql`: formats array, bool, and string in SQL style

You can define filters by yourself:
```julia
@filter repeat say_twice(txt) = txt*txt
@filter function greet(x)
    return x * "Hello"
end
```
Then you can use `repeat` and `greet` in you template. See also: [`@filter`](@ref).

!!! warning "Do not use the same name for a filter and a variable name"
    When a Template builds renderer, `build_renderer` function finds all the undefined symbol which should be passed by user.
    If the name is already used for a filter, the function can't recognize the variable undefined, and then it returns unexpected output.

## Comment
To comment out parts of template, use comment block which set to `{# #}` by default:
```
{#
    This is comment.
    These lines are just ignored and removed
#}
```
Comment blocks can be nested like this:
```
{#
   Comments look something like this: {# foo #}
#}
```
Ofcourse this sample render empty.

## White Space Control
OteraEngine has option to control spaces which is named `lstrip_blocks` and `trim_blocks`.
If `lstrip_blocks` is enabled, spaces from start of line behind the block is removed.
Template:
```
{% for i in 1:3 %}
    Hello {{ i }}
{% end %}
```
Without `lstrip blocks`:
```html
<div>

        Hello 1

        Hello 2

        Hello 3

</div>
```
With `lstrip_blocks`(you can't see the difference. please try selecting the text):
```html
<div>

        Hello 1

        Hello 2

        Hello 3

</div>
```
If `trim_blocks` is enabled, the (only) first newline after the block is removed.
Without `trim_blocks`(`lstrip_blocks` is disabled):
```html
<div>
            Hello 1
            Hello 2
            Hello 3
    </div>
```
With `trim_blocks` and `lstrip_blocks`:
```html
<div>
        Hello 1
        Hello 2
        Hello 3
</div>
```
But, sometimes these options aren't perfect(like macro), and it's annoying to set these options all the time. So, you can use `autospace` option which automatically enables these options and remove extra spaces from macro.

And other way to control white spaces is to add `+` and `-` to the both ends of blocks. This works partially even if space control options are enabled.
`+` means that the block does nothing about space. And `-` means that the block removes all the spaces.
This is useful when you want to put items in a row with `for` block:
```html
<div>
    {% for i in 1 : 10 -%}
    {{i}}
    {%- end %}
</div>
```
Output:
```html
<div>
    12345678910
</div>
```

## Escaping
It is important to apply HTML escaping in order to prevent XSS. So, `autoescape` is set to `true` by default.
If you want to escape manually, you can disable this option, and use `e` or `escape` filter into expression blocks:
```html
<div>
    {{ value |> e }}
</div>
```
Where `value` is `<script>This is injection attack</script>`
```html
<div>
    &lt;script&gt;This is injection attack&lt;/script&gt;
</div>
```

## Raw Text
Sometimes it is neccessary to ignore blocks and recognize it raw text. Then, you should use `raw` block:
```
{% raw %}
This is test for raw block
{% you can write anything inside raw block %}
{% endraw %}
```

## Template Inheritance
### Include
You can include template with `{% include "(template filename)" %}` code block. This is the tiny example:
```
#=This is the included template(test2.html)=#
Hello everyone! My name is watasu.
```
```
#=This is the main template=#
{% include "test2.html" %}
Today, I'd like to introduce OteraEngine.jl
```

!!! warning "Template filename have to be enclosed with double quotation mark"
    Template filename have to be like this: `"test.html"`. Otherwise, parser returns error.

This code block is also available inside the `{% block %}` explained in next section.

### Extends
When you build large web app with OteraEngine, you may want to use "template of template". This is possible with `{% extends %}` code block.
This code block have to be located at the top of the document, otherwise ignored. This is the example:
```html
#=This is the base template(test2.html)=#
<!DOCTYPE html>
<html>
    <head>
        <title>test for extends</title>
    </head>
    <body>
        <div>
            {% block body %}
            {% endblock %}
        </div>
    </body>
</html>
```
```html
#=This is the main template=#
{% extends "test2.html" %}
{% block body %}
            <h1>hello</h1>
            <div>
                <p>some content here.</p>
            </div>
{% endblock %}
```
```html
#=Output=#
<!DOCTYPE html>
<html>
    <head>
        <title>test for extends</title>
    </head>
    <body>
        <div>
            <h1>hello</h1>
            <div>
                <p>some content here.</p>
            </div>
        </div>
    </body>
</html>
```

!!! warning "Template filename have to be enclosed with double quotation mark"
    Template filename have to be like this: `"test.html"`. Otherwise, parser returns error.

If you write `{% extends (template filename) %}` in main template, parser will use `(template filename)` as the base template.
And, you can write blocks in the main template with `{% block (block name) %}` and `{% endblock %}`.
When you build the very complex templates, nested extended templates must be useful.

### Nested Extens
When you build very complex templates, nested extended templates must be useful. This is the exmaple:
```html
#nestedextends1.html
{% extends "nestedextends2.html" %}

{% block content2 %}
        <p>Hello</p>
{% endblock %}
```
```html
#nestedextends2.html
{% extends "nestedextends3.html" %}

{% block content %}
        <h1>Title</h1>
        {% block content2 %}{% endblock %}
{% endblock %}
```
```html
#nestedextends3.html
<!DOCTYPE html>
<html>
    <head>
        <title>Nested Extends Test</title>
    </head>
    <body>
        This is the test for nested extends blocks. Here it is:
        {% block content %}
        {% endblock %}
    </body>
</html>
```
```html
#Output
<!DOCTYPE html>
<html>
    <head>
        <title>Nested Extends Test</title>
    </head>
    <body>
        This is the test for nested extends blocks. Here it is:
        <h1>Title</h1>
        <p>Hello</p>
    </body>
</html>
```
In this case, `nestedextends1.html` loads `nestedextends2.html`, and `nestedextends2.html` loads `nestedextends3.html`. There is no limit in the depth of nest.

### Block Inheritance
Blocks defined in parent(even in ancestors) is inherited with `super()`:
```html
# Grand Parent Template("grand.html")
<div>
    {% block body %}
    Hello Grand Parents
    {%- endblock %}
</div>
# Parent Template("parent.html")
{% extends "grand.html" %}
{% block body %}
Hello Parent
{% endblock %}
# Child Template
{% extends "parent.html" %}
{% block body %}
    Hello Child
    {{ super() }}{{ super.super() }}
{% endblock %}
```
The, we get this:
```html
<div>
    Hello Child
    Hello Parent
    Hello Grand Parents
</div>
```
When you want to inherite ancestor's block, you just need to add `super.` before `super()`.

## Control Flow
There are 4 blocks available for control flow. `if`, `for`, `let` and `set`.
These blocks are converted into the Julia code directly, and the syntax is completely same with Julia.

### If
`if` block adds the text when condition is true:
```
{% if (condition1) %}
    This is `if` block
{% elseif (condition2) %}
    This is `elseif` block
{% else %}
    This is `else` block
{% end %}
```

### For
`for` block repeats the text for specified times:
```
{% for i in 1:5 %}
    Hello {{ i }}
{% end %}
```
you can use variables defined inside of `for` block.

### Let
`let` block creates local variables which has the scope inside of this block:
```
{% let name = "Julia" %}
    Hello {{ name }}
{% end %}
```
This is equal to `with` block in Jinja2.

### Set
`set` block is converted into `global (variable) = (value)`. So you can use variables in every control blocks(not in expression blocks now).

## Macro
Macro is similar to function in Programming Language. In fact, OteraEngine converts macros into Julia function internally. You can define different macro with the sae name as long as the arguments are different. If you write some functions with same name and arguments, macros other than the first would be ignored(In other words, method overwriting isn't supported).
This is the example:
```html
{% macro input(name, value="", type="text", size=20) %}
    <input type="{{ type }}" name="{{ name }}" value="{{
        value|>e }}" size="{{ size }}">
{% endmacro %}

<html>
    <head><title>MyPage</title></head>
    <body>
        {{ input("test") }}
    </body>
</html>
```
You should note that macro emits extra white space when you don't use any white space control options. So, it is strongly recommended to use `autospace` when you use macros.

## Import Macros
Sometimes you need the macro defined the different template. In such case, you should use `import` and `from`.
`import` adds an external template into the namespace with alias like this:
```html
{% import "import2.html" as base %}

<html>
    <head><title>MyPage</title></head>
    <body>
        {{ base.input("test") }}
    </body>
</html>
```
This template generate the same output as the previous example.

`from` is more flexible. You can directly add external macros into the namespace, and you can give it a different name or just add it as is.
```html
{% from "from2.html" import input, title as alias %}

<html>
    <head><title>MyPage</title></head>
    <body>
        {{ alias("Test") }}
        {{ input("test") }}
    </body>
</html>
```

## Configurations
there are 11 configurations:
- `control_block_start`: the string at the start of tmp code blocks.
- `control_block_end` : the string at the end of tmp code blocks.
- `expression_block_start` : the string at the start of expression blocks.
- `expression_block_end` : the string at the end of expression blocks.
- `comment_block_start`: the string at the start of comment blocks.
- `comment_block_end`: the string at the end of comment block.
- `autospace`: the option to control space automatically.
- `lstrip_blocks`: the option to strip left spaces.
- `trim_blocks`: the option to remove the first newline after blocks.
- `autoescape`: the option to automatically escape expression blocks
- `dir`: the working directory which `include` and `extends` statements refers to
and the default configuration is this:
```
"control_block_start"=>"{%",
"control_block_end"=>"%}",
"expression_block_start"=>"{{",
"expression_block_end"=>"}}",
"comment_block_start" => "{#",
"comment_block_end" => "#}",
"autospace" => true,
"lstrip_blocks" => false,
"trim_blocks" => false,
"autoescape" => true,
"dir" => pwd()
```
These configuration can be changed by using `config` argument of `Template()` like this:
```
tmp = Template("config.html", config=Dict("control_block_start"=>"{$", "control_block_end"=>"$}"))
```
configurations can be loaded from TOML file. You don't have to specify all the configurations(The rest uses the default settings).

!!! warning "Same character settings"
    If you set the same characters into the different items, tokenizer won't be able to work. Even if they matches partially, tokenizer may not work. Moreover, `ParserConfig` checks the match between `~_start` and `~_end`, but others not.
