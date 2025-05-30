[![License LGPLv3][LGPLv3 badge]][LGPLv3]
[![License ASL 2.0][ASL 2.0 badge]][ASL 2.0]
[![Pipeline][GitHub Acitons badge]][GitHub Acitons link]
[![Maven Central][Maven Central badge]][Maven]

## Read me first

This project, as of version 1.4, is licensed under both LGPLv3 and ASL 2.0. See
file LICENSE for more details. Versions 1.3 and lower are licensed under LGPLv3
only.

**Note the "L" in "LGPL". LGPL AND GPL ARE QUITE DIFFERENT!**

## Fork

This library was forked by [gravity9](https://www.gravity9.com) from the
original [repository](https://github.com/java-json-tools/json-patch)
to maintain and extend it as the original library is no longer supported.

## What this is

This is an implementation of [RFC 6902 (JSON Patch)](http://tools.ietf.org/html/rfc6902) and [RFC
7386 (JSON
Merge Patch)](http://tools.ietf.org/html/rfc7386) written in Java,
which uses [Jackson](https://github.com/FasterXML/jackson-databind) (2.x) at its core.

Its features are:

* {de,}serialization of JSON Patch and JSON Merge Patch instances with Jackson;
* full support for RFC 6902 operations, including `test`;
* JSON "diff" (RFC 6902 only) with operation factorization.
* support for `JsonPointer` and `JsonPath`

## Versions

The current version is **2.0.2**. See file `RELEASE-NOTES.md` for details of releases before 1.11.

## Using it in your project

With Gradle:

```groovy
dependencies {
  compile(group: "com.gravity9", name: "json-patch-path", version: "yourVersionHere");
}
```

With Maven:

```xml
<dependency>
  <groupId>com.gravity9</groupId>
  <artifactId>json-patch-path</artifactId>
  <version>2.0.2</version>
</dependency>
```

Versions before 1.10 are available at `groupId` `com.github.fge` and `artifactId` `json-patch`.
Versions before 1.13 are available at `groupId` `com.github.java-json-tools` and `artifactId` `json-patch`.

## JSON "diff" factorization

When computing the difference between two JSON texts (in the form of `JsonNode` instances), the diff
will factorize value removals and additions as moves and copies.

For instance, given this node to patch:

```json
{ "a": "b" }
```

in order to obtain:

```json
{ "c": "b" }
```

the implementation will return the following patch:

```json
[ { "op": "move", "from": "/a", "path": "/c" } ]
```

It is able to do even more than that. See the test files in the project.

## Note about the `test` operation and numeric value equivalence

RFC 6902 mandates that when testing for numeric values, however deeply nested in the tested value,
a test is successful if the numeric values are _mathematically equal_. That is, JSON texts:

```json
1
```

and:

```json
1.00
```

must be considered equal.

This implementation obeys the RFC; for this, it uses the numeric equivalence of
[jackson-coreutils](https://github.com/fge/jackson-coreutils).

## Sample usage

### JSON Patch

You have two choices to build a `JsonPatch` instance: use Jackson deserialization, or initialize one
directly from a `JsonNode`. Examples:

```java
// Using Jackson
final ObjectMapper mapper = new ObjectMapper();
final InputStream in = ...;
final JsonPatch patch = mapper.readValue(in, JsonPatch.class);
// From a JsonNode
final JsonPatch patch = JsonPatch.fromJson(node);
```

You can then apply the patch to your data:

```java
// orig is also a JsonNode
final JsonNode patched = patch.apply(orig);
```

### JSON diff

The main class is `JsonDiff`. It returns the patch as a `JsonPatch` or as a `JsonNode`. Sample usage:

```java
final JsonPatch patch = JsonDiff.asJsonPatch(source, target);
final JsonNode patchNode = JsonDiff.asJson(source, target);
```

It's possible to ignore fields in Json Diff. List of ignored fields should be specified as JsonPointer 
or JsonPath paths. If ignored field does not exist in target or source object, it's ignored.

```java
final List<String> fieldsToIgnore = new ArrayList<>();
fieldsToIgnore.add("/id");
fieldsToIgnore.add("$.cars[-1:]");
final JsonPatch patch = JsonDiff.asJsonPatch(source, target, fieldsToIgnore);
final JsonNode patchNode = JsonDiff.asJson(source, target, fieldsToIgnore);
```

**Important note**: the API offers **no guarantee at all** about patch "reuse";
that is, the generated patch is only guaranteed to safely transform the given
source to the given target. Do not expect it to give the result you expect on
another source/target pair!

### JSON Merge Patch

As for `JsonPatch`, you may use either Jackson or "direct" initialization:

```java
// With Jackson
final JsonMergePatch patch = mapper.readValue(in, JsonMergePatch.class);
// With a JsonNode
final JsonMergePatch patch = JsonMergePatch.fromJson(node);
```

Applying a patch also uses an `.apply()` method:

```java
// orig is also a JsonNode
final JsonNode patched = patch.apply(orig);
```

## Examples of JSON Patch operations with JSON Pointer

### Add operation

* Add field `a` with value `1` to object  
`{ "op": "add", "path": "/a", "value": 1 }`  
  Before:
    ```json
    {
      "b": "test2"
    }
    ```

  After:
    ```json
    {
      "a": 1,
      "b": "test2"
    }
    ```
    <br />

* Add element with value `1` at the end of array with name `array`  
`{ "op": "add", "path": "/array/-", "value": 1 }`  
  Before:
    ```json
    {
      "array": [0, 1, 2, 3]
    }
    ```

  After:
    ```json
    {
      "array": [0, 1, 2, 3, 1]
    }
    ```
    <br />

* Add element with value `1` at the specific index of array with name `array`  
`{ "op": "add", "path": "/array/2", "value": 1 }`  
  Before:
    ```json
    {
      "array": [0, 1, 2, 3]
    }
    ```

  After:
    ```json
    {
      "array": [0, 1, 1, 2, 3]
    }
    ```
    <br />

* Add element with name `b` into inner object  
`{ "op": "add", "path": "/obj/inner/b", "value": [ 1, 2 ] }`  
  Before:
    ```json
    {
      "obj": {
        "inner": {
          "a": "test"
        }   
      }
    }
    ```

  After:
    ```json
    {
      "obj": {
        "inner": {
          "a": "test",
          "b": [1, 2]
        }   
      }
    }
    ```
    <br />
* If element with name `a` exists, then `add` operation overrides it.  
`{ "op": "add", "path": "/a", "value": 1 }`

    Before:
    ```json
    {
      "a": 0
    }
    ```
    
    After:
    ```json
    {
      "a": 1
    }
    ```
    <br />

### Add if not exists
It's possible to add element to JsonNode if it does not exist using JsonPath expressions [see more examples of JsonPath](#jsonpath-examples)
* Add `color` field to `bicycle` object if it doesn't exist  
`{ "op": "add", "path": "$.store.bicycle[?(!@.color)].color", "value": "red" }`  

    Before:
    ```json
    {
      "store": {
        "bicycle": {
          "price": 19.95
        }
      }
    }
    ```
  
    After:
    ```json
    {
      "store": {
        "bicycle": {
          "price": 19.95,
          "color": "red"
        }
      }
    }
    ```
* Add value for `color` field to `bicycle` object if it is equal to `null`  
  `{ "op": "add", "path": "$.store.bicycle[?(@.color == null)].color", "value": "red" }`  

  Before:
    ```json
    {
      "store": {
        "bicycle": {
          "price": 19.95,
          "color": null
        }
      }
    }
    ```

  After:
    ```json
    {
      "store": {
        "bicycle": {
          "price": 19.95,
          "color": "red"
        }
      }
    }
    ```

* Add field `pages` to `book` array if `book` does not contain this field, or it is equal to `null`   
  `{ "op": "add", "path": "$..book[?(!@.pages || @.pages == null)].pages", "value": 250 }`

  Before:
    ```json
    {
        "store": {
            "book": [
                {
                    "category": "reference",
                    "author": "Nigel Rees",
                    "title": "Sayings of the Century",
                    "price": 8.95
                },
                {
                    "category": "fiction",
                    "author": "Herman Melville",
                    "title": "Moby Dick",
                    "isbn": "0-553-21311-3",
                    "price": 8.99,
                    "pages": null
                },
                {
                    "category": "fiction",
                    "author": "J.R.R. Tolkien",
                    "title": "The Lord of the Rings",
                    "isbn": "0-395-19395-8",
                    "price": 22.99,
                    "pages": 100
                }
            ]
        }
    }
    ```

  After:
     ```json
    {
        "store": {
            "book": [
                {
                    "category": "reference",
                    "author": "Nigel Rees",
                    "title": "Sayings of the Century",
                    "price": 8.95,
                    "pages": 250
                },
                {
                    "category": "fiction",
                    "author": "Herman Melville",
                    "title": "Moby Dick",
                    "isbn": "0-553-21311-3",
                    "price": 8.99,
                    "pages": 250
                },
                {
                    "category": "fiction",
                    "author": "J.R.R. Tolkien",
                    "title": "The Lord of the Rings",
                    "isbn": "0-395-19395-8",
                    "price": 22.99,
                    "pages": 100
                }
            ]
        }
    }
    ```

### Remove operation

* Remove element with name `a`  
`{ "op": "remove", "path": "/a" }`   
    Before:
    ```json
    {
      "a": "test",
      "b": "test2"
    }
    ```
    
    After:
    ```json
    {
      "b": "test2"
    }
    ```
    <br />

* Remove element from array with name `list` at index `2`  
`{ "op": "remove", "path": "/list/2" }`  
    Before:
    ```json
    { 
      "a": "test",
      "list": [0, 1, 2, 3, 4]
    }
    ```
  
    After:
    ```json
    {
      "a": "test",
      "list": [0, 1, 3, 4]
    }
    ```
### Replace operation

* Replace value for element with name `a` to `new-value`  
`{ "op": "replace", "path": "/a", "value": "new-value"}`  

    Before:
    ```json
    {
      "a": "test",
      "b": "test2"
    }
    ```
    
    After:
    ```json
    {
      "a": "new-value",
      "b": "test2"
    }
    ```
    <br />

* Replace value with `new-value` for 2nd element in array with name `array`  
`{ "op": "replace", "path": "/array/2", "value": "new-value"}`  
Before:
    ```json
    {
      "a": "test",
      "array": ["test0", "test1", "test2"]
    }
    ```

    After:
    ```json
    {
      "a": "test",
      "array": ["test0", "test1", "new-value"]
    }
    ```
  
### Copy operation

* Copy value from filed `a` to field `b` which does not exist  
`{ "op": "copy", "from": "/a", "path": "/b" }`

  Before:
  ```json
  {
    "a": "test"
  }
  ```
  
  After:
  ```json
  {
    "a": "test",
    "b": "test"
  }
  ```

* Copy value from filed `a` to field `b` which exists - value will be updated  
  `{ "op": "copy", "from": "/a", "path": "/b" }`  

  Before:
  ```json
  {
    "a": "test",
    "b": "old value"
  }
  ```

  After:
  ```json
  {
    "a": "test",
    "b": "test"
  }
  ```
  
* Copy first element of array to the end of array  
  `{ "op": "copy", "from": "/array/0", "path": "/array/-" }`

  Before:
  ```json
  {
    "array": [0, 1, 2],
    "b": "old value"
  }
  ```

  After:
  ```json
  {
    "array": [0, 1, 2, 0],
    "b": "old value"
  }
  ```
  
### Move operation

* Move value from field `a` to field `b`  
`{ "op": "move", "from": "/a", "path": "/b" }`

  Before:
  ```json
  {
    "a": 1,
    "b": 2
  }
  ```
  
  After:
  ```json
  {
    "b": 1
  }
  ```
  <br />

* Move first element of an array to the end of array
  `{ "op": "move", "from": "/array/0", "path": "/array/-" }`

  Before:
  ```json
  {
    "array": [1, 2, 3]
  }
  ```

  After:
  ```json
  {
    "array": [2, 3, 1]
  }
  ```
  <br />

### Test operation
* Check if field `a` has value `test-value`  
  `{ "op": "test", "path": "/a", "value": "test-value" }`


## JsonPath examples

JsonPath is supported in all operations (`add`, `remove`, `copy`, `replace`, `move`, `test`).

Examples of JsonPath:

* `$.store.bicycle.price` - Get price of bicycle in store
* `$.store.book[*].pages` - Get pages from all books in store
* `$..book[*].pages` - Get pages from all books which are a descent of root node
* `$.store.book[-1:].pages` - Get pages from last book in store
* `$.store.book[:2].pages` - Get pages of first two books in store
* `$.store.book[?(@.author=='J.R.R. Tolkien')].pages` - Get pages of books written by J.R.R. Tolkien
* `$..book[?(@.isbn)].pages` - Get pages of books which contain `isbn` property
* `$..book[?(!@.isbn)].pages` - Get pages of books which do not contain `isbn` property
* `$..book[?(@.price < 8.99)].pages` - Get pages of books which price is lower than `8.99`
* `$..book[?(@.author =~ /.*Tolkien/i)].pages` - Pages of books whose author name ends with Tolkien (case-insensitive).
* `$..book[?(@.category == 'fiction' || @.category == 'reference')]` - All books in `fiction` or `reference` category
* `$..book[?(@.category in ['fiction', 'reference'])]` - All books in `fiction` or `reference` category
* `$..book[?(@.category nin ['fiction'])]` - All books not in `fiction` category
* `$..book[?(@.category=='fiction' && @.price < 10)].pages` - List of pages of books in `fiction` category and price lower than `10`
* `$..book[?(@.tags subsetof ['tag1', 'tag2'])]` - All books with list of tags which is subset of `['tag1', 'tag2']`
* `$..book[?(@.tags contains 'tag2')]` - All books with list of tags containing `tag2`
* `$..book[?(@.tags size 2)]` - All books with list of tags containing exactly 2 elements
* `$..book[?(@.tags empty true)]` - All books with empty list of tags
* `$..book[?(@.tags empty false)]` - All books with not empty list of tags

### Limitations

* `$..book[(@.length-1)].title` - not supported. Use `$..book[-1:].title` instead.

[LGPLv3 badge]: https://img.shields.io/:license-LGPLv3-blue.svg

[LGPLv3]: http://www.gnu.org/licenses/lgpl-3.0.html

[ASL 2.0 badge]: https://img.shields.io/:license-Apache%202.0-blue.svg

[ASL 2.0]: http://www.apache.org/licenses/LICENSE-2.0.html

[GitHub Acitons badge]: https://github.com/gravity9-tech/json-patch-path/actions/workflows/push-workflow.yaml/badge.svg?branch=master

[GitHub Acitons link]: https://github.com/gravity9-tech/json-patch-path/actions/workflows/push-workflow.yaml

[Maven Central badge]: https://img.shields.io/maven-central/v/com.gravity9/json-patch-path.svg

[Maven]: https://search.maven.org/artifact/com.gravity9/json-patch-path
