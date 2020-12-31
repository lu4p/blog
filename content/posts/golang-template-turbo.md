---
title: "Understanding HTML templates in Go (golang)"
date: 2020-12-30T03:28:14+01:00
tags: ["golang", "templates"]
---


*I use [Echo](https://echo.labstack.com/) in this post, but you could also use plain [net/http](https://pkg.go.dev/net/http), or any of the other awesome web frameworks for Go, see a list [here](https://go.dev/solutions/cloud/#get-started).*

The Go standard library provides a [html/template](https://pkg.go.dev/html/template) package, for dynamically rendering HTML, it is built on top of [text/template](https://pkg.go.dev/text/template)

Go templates are normal HTML files, with "Actions" (data evaluations or control structures), 
these "Actions" are delimited via `{{`and`}}`.

A template is executed via applying a data structure to it, the data structure is referenced as a dot `.` in the template.

### Parsing Templates
Templates can be defined as string literals in a Go file, but it's easier for organizing your templates and utilizing HTML support in your editor, to write your templates as separate files.

This `templates, _ := template.ParseGlob("templates/*.html")`,
parses all `.html` files in the `templates` directory.

### Writing Templates
This is an example template:

```go-html-template
{{ template "head" . }}

{{ link "/foo" "Foo"}}
<p class="bg-gray-100">{{.test}}</p>
<p>{{toString .slice }}</p>


{{ template "foot" }}
```

**But what does this do exactly?**

`{{ template "head" . }}` embeds the `head` template from elsewhere and passes all template data to  it (indicated by the dot).

`{{ link "/foo" "Foo"}}` calls the function link with the parameters `"/foo"` and `"Foo"`, in plain Go this would be `link("/foo", "Foo")`.

`{{.test}}` outputs the value of `.test`.

`{{ toString .slice }}` calls the function toString with the parameter `.slice`.

`{{ template "foot" }}` embeds the `foot` template from elsewhere, no data is passed (notice the missing dot).

### Passing data to a template (rendering)
```go
func root(c echo.Context) error {
	return c.Render(200, "index.html", map[string]interface{}{
		"title": "Root",
		"test":  "Hello, world!",
		"slice": []int{1, 2, 3},
	})
}
```

A template is executed via supplying data for "dot" (`.`). In Echo this is done with `c.Render` inside a handler function. 

You pass a HTTP status code (200), the name of the template to execute (index.html), and the data (in this case a map literal).

To access the data from the template you can use "dot", an element of the map can be accessed via `.key`, for structs it's similarly `.field`, methods of a data type can be called via `.Method`.

### Registering Functions
It is nice to execute functions right in the template, for control flow e.g. "Is a user logged in?", or to render something.


A function can be added to a template as follows:
```go
t := template.New("new-template")
t.Funcs(template.FuncMap{
	"email": func() string {
		return "example@example.com"
	},
	"anotherFunc": anotherFunc,
})
```

In your template you can then call the `email` function to show the email of the currently logged in user.
```html
<p>Your email address: {{ email }}</p>
```

### Defining templates
You can define reusable blocks of HTML via wrapping the code in a "define" block, see the following example:
```go-html-template
{{ define "p"}}
<p>{{.}}</p>
{{ end }}
```

This can then be used in another template/ file via:
```go-html-template
{{ template "p" "Some content"}}
```

And will output:
```html
<p>Some content</p>
```

A more real world use case would be the following, which defines a "head" and a "foot" template:
```go-html-template
# templates/base.html
{{ define "head" }}
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{.title}}</title>
    <link rel="stylesheet" href="/dist/main.css">
    <script src="/dist/main.js" defer></script>
</head>

<body>
    <div class="container mx-auto">
        <h1>{{.title}}</h1>
        {{ end }}

        {{ define "foot" }}
    </div>
</body>

</html>
{{ end }}
```

These templates were already used in the example template from the beginning:
```go-html-template
# templates/index.html
{{ template "head" . }}

{{ link "/foo" "Foo"}}
<p class="bg-gray-100">{{.test}}</p>
<p>{{toString .slice }}</p>


{{ template "foot" }}
```

### There is more
Go templates also support additional functionality not covered in this post, notably:
- `if`/ `else` blocks for conditional rendering
- `range` for looping over a slice, map, array or channel
- `with` only rendering if a value exists

See https://pkg.go.dev/text/template, for more information.


## Sample Project
I wrote a quick sample project which also includes additional functionality, it can be found [here](https://github.com/lu4p/go-template-turbo-sample).

The sample project also uses:
- [Turbo](https://turbo.hotwire.dev/), part of [Hotwire](https://hotwire.dev/) a new approach from Basecamp for writing modern web applications without much JavaScript. I will maybe do a separate post about Turbo in the Future.
- [tailwindcss](https://tailwindcss.com/), makes HTML look nice.
- [webpack](https://webpack.js.org/), for packing JS and CSS into single files, with minimization enabled, setup to extract CSS to a separate file.
- [Air](https://github.com/cosmtrek/air), for hot reloading Go code and templates on change.