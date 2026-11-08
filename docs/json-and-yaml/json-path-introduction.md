# Introduction to JSONPath

JSONPath is a query syntax for selecting values from JSON documents. A path starts at the root (`$`) and follows object properties or array elements.

## Example JSON

```json
{
  "store": {
    "name": "Book Shop",
    "books": [
      { "title": "Kubernetes Basics", "price": 25 },
      { "title": "Cluster Operations", "price": 30 }
    ]
  }
}
```

## Selecting Values

| Expression | Selects |
| --- | --- |
| `$` | The entire document |
| `$.store.name` | The store name |
| `$.store.books[0].title` | The title of the first book |
| `$.store.books[1].price` | The price of the second book |

Bracket notation can also be used for property names:

```text
$.store["name"]
```

JSONPath is useful when a command or tool returns a large JSON document and you only need specific fields. Kubernetes' `kubectl` supports its own JSONPath output syntax, covered in the Kubernetes examples.
