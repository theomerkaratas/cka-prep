# JSONPath Wildcards

The wildcard (`*`) matches all properties or elements at a position in a JSONPath expression. It is useful when the names or indexes of the values are not known in advance.

Using the book data from [Introduction to JSONPath](json-path-introduction.md):

```text
$.store.books[*].title
```

This selects the `title` property from every element in the `books` array.

Wildcards can also select all values in an object:

```json
{
  "metadata": {
    "name": "nginx",
    "namespace": "default"
  }
}
```

```text
$.metadata.*
```

This selects both values under `metadata`. The exact output format depends on the JSONPath implementation that evaluates the expression.
