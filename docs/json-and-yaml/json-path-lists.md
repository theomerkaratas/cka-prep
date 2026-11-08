# JSONPath Lists

JSON arrays are ordered lists. Use an index in square brackets to select one element; indexes start at `0`.

Given this array:

```json
{
  "names": ["api", "worker", "database"]
}
```

| Expression | Result |
| --- | --- |
| `$.names[0]` | `api` |
| `$.names[1]` | `worker` |
| `$.names[2]` | `database` |

Use a wildcard to select a value from every list item. For example, `$.store.books[*].title` selects every book title from the data in [Introduction to JSONPath](json-path-introduction.md).

Use indexes only when the position is meaningful and stable. If you need values from every item, a wildcard or the JSONPath implementation's iteration feature is usually more appropriate.
