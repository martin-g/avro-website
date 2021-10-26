---
title: "C API"
linkTitle: "C API"
weight: 5
---

The current version of Avro is {{< avro_version >}}. The current version of `libavro` is `23:0:0`. This document was created 2021-10-21.

## 1. Introduction to Avro
Avro is a data serialization system.

Avro provides:

* Rich data structures.
* A compact, fast, binary data format.
* A container file, to store persistent data.
* Remote procedure call (RPC).

This document will focus on the C implementation of Avro. To learn more about Avro in general, visit the [Avro website](/).

## 2. Introduction to Avro C
```
    ___                      ______
   /   |_   ___________     / ____/
  / /| | | / / ___/ __ \   / /
 / ___ | |/ / /  / /_/ /  / /___
/_/  |_|___/_/   \____/   \____/
```
> 
> A C program is like a fast dance on a newly waxed dance floor by people carrying razors. — Waldi Ravens
>

The C implementation has been tested on MacOSX and Linux but, over time, the number of support OSes should grow. Please let us know if you’re using `Avro C` on other systems.

Avro depends on the [Jansson JSON parser](http://www.digip.org/jansson/), version 2.3 or higher. On many operating systems this library is available through your package manager (for example, `apt-get install libjansson-dev` on Ubuntu/Debian, and `brew install jansson` on Mac OS). If not, please download and install it from source.

The C implementation supports:

* binary encoding/decoding of all primitive and complex data types
* storage to an Avro Object Container File
* schema resolution, promotion and projection
* validating and non-validating mode for writing Avro data

The C implementation is lacking:

* RPC

To learn about the API, take a look at the examples and reference files later in this document.

We’re always looking for contributions so, if you’re a C hacker, please feel free to [submit patches to the project](TODO link to HOW TO CONTRIBUTE).

## 3. Error reporting
Most functions in the Avro C library return a single int status code. Following the POSIX `errno.h` convention, a status code of 0 indicates success. Non-zero codes indiciate an error condition. Some functions return a pointer value instead of an int status code; for these functions, a NULL pointer indicates an error.

You can retrieve a string description of the most recent error using the avro_strerror function:
```c
avro_schema_t  schema = avro_schema_string();
if (schema == NULL) {
    fprintf(stderr, "Error was %s\n", avro_strerror());
}
```
## 4. Avro values
Starting with version 1.6.0, the Avro C library has a new API for handling Avro data. To help distinguish between the two APIs, we refer to the old one as the `legacy` or `datum` API, and the new one as the `value` API. (These names come from the names of the C types used to represent Avro data in the corresponding API — `avro_datum_t` and `avro_value_t`.) The legacy API is still present, but it’s deprecated — you shouldn’t use the `avro_datum_t` type or the avro_datum_* functions in new code.

One main benefit of the new value API is that you can treat any existing C type as an Avro value; you just have to provide a custom implementation of the value interface. In addition, we provide a generic value implementation; “generic”, in this sense, meaning that this single implementation works for instances of any Avro schema type. Finally, we also provide a wrapper implementation for the deprecated avro_datum_t type, which lets you gradually transition to the new value API.

### 4.1. Avro value interface
You interact with Avro values using the value interface, which defines methods for setting and retrieving the contents of an Avro value. An individual value is represented by an instance of the avro_value_t type.

This section provides an overview of the methods that you can call on an avro_value_t instance. There are quite a few methods in the value interface, but not all of them make sense for all Avro schema types. For instance, you won’t be able to call avro_value_set_boolean on an Avro array value. If you try to call an inappropriate method, we’ll return an EINVAL error code.

Note that the functions in this section apply to all Avro values, regardless of which value implementation is used under the covers. This section doesn’t describe how to create value instances, since those constructors will be specific to a particular value implementation.

#### 4.1.1. Common methods
There are a handful of methods that can be used with any value, regardless of which Avro schema it’s an instance of:
```c
#include <stdint.h>
#include <avro.h>

avro_type_t avro_value_get_type(const avro_value_t *value);
avro_schema_t avro_value_get_schema(const avro_value_t *value);

int avro_value_equal(const avro_value_t *v1, const avro_value_t *v2);
int avro_value_equal_fast(const avro_value_t *v1, const avro_value_t *v2);

int avro_value_copy(avro_value_t *dest, const avro_value_t *src);
int avro_value_copy_fast(avro_value_t *dest, const avro_value_t *src);

uint32_t avro_value_hash(avro_value_t *value);

int avro_value_reset(avro_value_t *value);
```

The `get_type` and `get_schema` methods can be used to get information about what kind of Avro value a given avro_value_t instance represents. (For `get_schema`, you borrow the value’s reference to the schema; if you need to save it and ensure that it outlives the value, you need to call avro_schema_incref on it.)

The `equal` and `equal_fast` methods compare two values for equality. The two values do not have to have the same value implementations, but they do have to be instances of the same schema. (Not equivalent schemas; the same schema.) The equal method checks that the schemas match; the equal_fast method assumes that they do.

The copy and copy_fast methods copy the contents of one Avro value into another. (Where possible, this is done without copying the actual content of a bytes, string, or fixed value, using the avro_wrapped_buffer_t functions described in the next section.) Like equal, the two values must have the same schema; copy checks this, while copy_fast assumes it.

The hash method returns a hash value for the given Avro value. This can be used to construct hash tables that use Avro values as keys. The function works correctly even with maps; it produces a hash that doesn’t depend on the ordering of the elements of the map. Hash values are only meaningful for comparing values of exactly the same schema. Hash values are not guaranteed to be consistent across different platforms, or different versions of the Avro library. That means that it’s really only safe to use these hash values internally within the context of a single execution of a single application.

The reset method “clears out” an +avro_value_t instance, making sure that it’s ready to accept the contents of a new value. For scalars, this is usually a no-op, since the new value will just overwrite the old one. For arrays and maps, this removes any existing elements from the container, so that we can append the elements of the new value. For records and unions, this just recursively resets the fields or current branch.

#### 4.1.2. Scalar values
The simplest case is handling instances of the scalar Avro schema types. In Avro, the scalars are all of the primitive schema types, as well as enum and fixed — i.e., anything that can’t contain another Avro value. Note that we use standard C99 types to represent the primitive contents of an Avro scalar.

To retrieve the contents of an Avro scalar, you can use one of the getter methods:
```c
#include <stdint.h>
#include <stdlib.h>
#include <avro.h>

int avro_value_get_boolean(const avro_value_t *value, int *dest);
int avro_value_get_bytes(const avro_value_t *value,
                         const void **dest, size_t *size);
int avro_value_get_double(const avro_value_t *value, double *dest);
int avro_value_get_float(const avro_value_t *value, float *dest);
int avro_value_get_int(const avro_value_t *value, int32_t *dest);
int avro_value_get_long(const avro_value_t *value, int64_t *dest);
int avro_value_get_null(const avro_value_t *value);
int avro_value_get_string(const avro_value_t *value,
                          const char **dest, size_t *size);
int avro_value_get_enum(const avro_value_t *value, int *dest);
int avro_value_get_fixed(const avro_value_t *value,
                         const void **dest, size_t *size);
```
For the most part, these should be self-explanatory. For bytes, string, and fixed values, the pointer to the underlying content is const — you aren’t allowed to modify the contents directly. We guarantee that the content of a string will be NUL-terminated, so you can use it as a C string as you’d expect. The size returned for a string object will include the NUL terminator; it will be one more than you’d get from calling strlen on the content.

Also, for bytes, string, and fixed, the dest and size parameters are optional; if you only want to determine the length of a bytes value, you can use:
```c
avro_value_t  *value = /* from somewhere */;
size_t  size;
avro_value_get_bytes(value, NULL, &size);
```
To set the contents of an Avro scalar, you can use one of the setter methods:
```c
#include <stdint.h>
#include <stdlib.h>
#include <avro.h>

int avro_value_set_boolean(avro_value_t *value, int src);
int avro_value_set_bytes(avro_value_t *value,
                         void *buf, size_t size);
int avro_value_set_double(avro_value_t *value, double src);
int avro_value_set_float(avro_value_t *value, float src);
int avro_value_set_int(avro_value_t *value, int32_t src);
int avro_value_set_long(avro_value_t *value, int64_t src);
int avro_value_set_null(avro_value_t *value);
int avro_value_set_string(avro_value_t *value, const char *src);
int avro_value_set_string_len(avro_value_t *value,
                              const char *src, size_t size);
int avro_value_set_enum(avro_value_t *value, int src);
int avro_value_set_fixed(avro_value_t *value,
                         void *buf, size_t size);
```
These are also straightforward. For bytes, string, and fixed values, the set methods will make a copy of the underlying data. For string values, the content must be NUL-terminated. You can use set_string_len if you already know the length of the string content; the length you pass in should include the NUL terminator. If you call set_string, then we’ll use strlen to calculate the length.

For fixed values, the size must match what’s expected by the value’s underlying fixed schema; if the sizes don’t match, you’ll get an error code.

If you don’t want to copy the contents of a bytes, string, or fixed value, you can use the giver and grabber functions:
```c
#include <stdint.h>
#include <stdlib.h>
#include <avro.h>

typedef void (*avro_buf_free_t)(void *ptr, size_t sz, void *user_data);

int avro_value_give_bytes(avro_value_t *value, avro_wrapped_buffer_t *src);
int avro_value_give_string_len(avro_value_t *value, avro_wrapped_buffer_t *src);
int avro_value_give_fixed(avro_value_t *value, avro_wrapped_buffer_t *src);

int avro_value_grab_bytes(const avro_value_t *value, avro_wrapped_buffer_t *dest);
int avro_value_grab_string(const avro_value_t *value, avro_wrapped_buffer_t *dest);
int avro_value_grab_fixed(const avro_value_t *value, avro_wrapped_buffer_t *dest);

typedef struct avro_wrapped_buffer {
    const void  *buf;
    size_t  size;
    void (*free)(avro_wrapped_buffer_t *self);
    int (*copy)(avro_wrapped_buffer_t *dest,
                const avro_wrapped_buffer_t *src,
                size_t offset, size_t length);
    int (*slice)(avro_wrapped_buffer_t *self,
                 size_t offset, size_t length);
} avro_wrapped_buffer_t;

void
avro_wrapped_buffer_free(avro_wrapped_buffer_t *buf);

int
avro_wrapped_buffer_copy(avro_wrapped_buffer_t *dest,
                         const avro_wrapped_buffer_t *src,
                         size_t offset, size_t length);

int
avro_wrapped_buffer_slice(avro_wrapped_buffer_t *self,
                          size_t offset, size_t length);
```
The `give` functions give control of an existing buffer to the value. (You should not try to free the src wrapped buffer after calling this method.) The grab function fills in a wrapped buffer with a pointer to the contents of an Avro value. (You should free the dest wrapped buffer when you’re done with it.)

The avro_wrapped_buffer_t struct encapsulates the location and size of the existing buffer. It also includes several methods. The free method will be called when the content of the buffer is no longer needed. The slice method will be called when the wrapped buffer needs to be updated to point at a subset of what it pointed at before. (This doesn’t create a new wrapped buffer; it updates an existing one.) The copy method will be called if the content needs to be copied. Note that if you’re wrapping a buffer with nice reference counting features, you don’t need to perform an actual copy; you just need to ensure that the free function can be called on both the original and the copy, and not have things blow up.

The “generic” value implementation takes advantage of this feature; if you pass in a wrapped buffer with a give method, and then retrieve it later with a grab method, then we’ll use the wrapped buffer’s copy method to fill in the dest parameter. If your wrapped buffer implements a slice method that updates reference counts instead of actually copying, then you’ve got nice zero-copy access to the contents of an Avro value.

#### 4.1.3. Compound values
The following sections describe the getter and setter methods for handling compound Avro values. All of the compound values are responsible for the storage of their children; this means that there isn’t a method, for instance, that lets you add an existing avro_value_t to an array. Instead, there’s a method that creates a new, empty avro_value_t of the appropriate type, adds it to the array, and returns it for you to fill in as needed.

You also shouldn’t try to free the child elements that are created this way; the container value is responsible for their life cycle. The child element is guaranteed to be valid for as long as the container value is. You’ll usually define an avro_value_t in the stack, and let it fall out of scope when you’re done with it:
```c
avro_value_t  *array = /* from somewhere else */;

{
    avro_value_t  child;
    avro_value_get_by_index(array, 0, &child, NULL);
    /* do something interesting with the array element */
}
```

#### 4.1.4. Arrays
There are three methods that can be used with array values:
```c
#include <stdlib.h>
#include <avro.h>

int avro_value_get_size(const avro_value_t *array, size_t *size);
int avro_value_get_by_index(const avro_value_t *array, size_t index,
                            avro_value_t *element, const char **unused);
int avro_value_append(avro_value_t *array, avro_value_t *element,
                      size_t *new_index);
```
The get_size method returns the number of elements currently in the array. The get_by_index method fills in element to point at the array element with the given index. (You should use NULL for the unused parameter; it’s ignored for array values.)

The append method creates a new value, appends it to the array, and returns it in element. If new_index is given, then it will be filled in with the index of the new element.

#### 4.1.5. Maps
There are four methods that can be used with map values:
```c
#include <stdlib.h>
#include <avro.h>

int avro_value_get_size(const avro_value_t *map, size_t *size);
int avro_value_get_by_name(const avro_value_t *map, const char *key,
                           avro_value_t *element, size_t *index);
int avro_value_get_by_index(const avro_value_t *map, size_t index,
                            avro_value_t *element, const char **key);
int avro_value_add(avro_value_t *map,
                   const char *key, avro_value_t *element,
                   size_t *index, int *is_new);
```
The get_size method returns the number of elements currently in the map. Map elements can be retrieved either by their key (get_by_name) or by their numeric index (get_by_index). (Numeric indices in a map are based on the order that the elements were added to the map.) In either case, the method takes in an optional output parameter that let you retrieve the index associated with a key, and vice versa.

The add method will add a new value to the map, if the given key isn’t already present. If the key is present, then the existing value with be returned. The index parameter, if given, will be filled in the element’s index. The is_new parameter, if given, can be used to determine whether the mapped value is new or not.

#### 4.1.6. Records
There are three methods that can be used with record values:
```c
#include <stdlib.h>
#include <avro.h>

int avro_value_get_size(const avro_value_t *record, size_t *size);
int avro_value_get_by_index(const avro_value_t *record, size_t index,
                            avro_value_t *element, const char **field_name);
int avro_value_get_by_name(const avro_value_t *record, const char *field_name,
                           avro_value_t *element, size_t *index);
```
The get_size method returns the number of fields in the record. (You can also get this by querying the value’s schema, but for some implementations, this method can be faster.)

The get_by_index and get_by_name functions can be used to retrieve one of the fields in the record, either by its ordinal position within the record, or by the name of the underlying field. Like with maps, the methods take in an additional parameter that let you retrieve the index associated with a field name, and vice versa.

When possible, it’s recommended that you access record fields by their numeric index, rather than by their field name. For most implementations, this will be more efficient.

#### 4.1.7. Unions
There are three methods that can be used with union values:
```c
#include <avro.h>

int avro_value_get_discriminant(const avro_value_t *union_val, int *disc);
int avro_value_get_current_branch(const avro_value_t *union_val, avro_value_t *branch);
int avro_value_set_branch(avro_value_t *union_val,
                          int discriminant, avro_value_t *branch);
```
The get_discriminant and get_current_branch methods return the current state of the union value, without modifying which branch is currently selected. The set_branch method can be used to choose the active branch, filling in the branch value to point at the branch’s value instance. (Most implementations will be smart enough to detect when the desired branch is already selected, so you should always call this method unless you can guarantee that the right branch is already current.)

### 4.2. Creating value instances
Okay, so we’ve described how to interact with a value that you already have a pointer to, but how do you create one in the first place? Each implementation of the value interface must provide its own functions for creating avro_value_t instances for that class. The 10,000-foot view is to:

Get an implementation struct for the value implementation that you want to use. (This is represented by an avro_value_iface_t pointer.)

Use the implementation’s constructor function to allocate instances of that value implementation.

Do whatever you need to the value (using the avro_value_t methods described in the previous section).

Free the value instance, if necessary, using the implementation’s destructor function.

Free the implementation struct when you’re done creating value instances.

These steps use the following functions:
```c
#include <avro.h>

avro_value_iface_t *avro_value_iface_incref(avro_value_iface_t *iface);
void avro_value_iface_decref(avro_value_iface_t *iface);
```
Note that for most value implementations, it’s fine to reuse a single avro_value_t instance for multiple values, using the avro_value_reset function before filling in the instance for each value. (This helps reduce the number of malloc and free calls that your application will make.)

We provide a “generic” value implementation that will work (efficiently) for any Avro schema.

For most applications, you won’t need to write your own value implementation; the Avro C library provides an efficient “generic” implementation, which supports the full range of Avro schema types. There’s a good chance that you just want to use this implementation, rather than rolling your own. (The primary reason for rolling your own would be if you want to access the elements of a compound value using C syntax — for instance, translating an Avro record into a C struct.) You can use the following functions to create and work with a generic value implementation for a particular schema:
```c
#include <avro.h>

avro_value_iface_t *avro_generic_class_from_schema(avro_schema_t schema);
int avro_generic_value_new(const avro_value_iface_t *iface, avro_value_t *dest);
void avro_generic_value_free(avro_value_t *self);
```
Combining all of this together, you might have the following snippet of code:
```c
avro_schema_t  schema = avro_schema_long();
avro_value_iface_t  *iface = avro_generic_class_from_schema(schema);

avro_value_t  val;
avro_generic_value_new(iface, &val);

/* Generate Avro longs from 0-499 */
int  i;
for (i = 0; i < 500; i++) {
    avro_value_reset(&val);
    avro_value_set_long(&val, i);
    /* do something with the value */
}

avro_generic_value_free(&val);
avro_value_iface_decref(iface);
avro_schema_decref(schema);
```

## 5. Reference Counting
Avro C does reference counting for all schema and data objects. When the number of references drops to zero, the memory is freed.

For example, to create and free a string, you would use:
```c
avro_datum_t string = avro_string("This is my string");

...
avro_datum_decref(string);
```
Things get a little more complicated when you consider more elaborate schema and data structures.

For example, let’s say that you create a record with a single string field:
```c
avro_datum_t example = avro_record("Example");
avro_datum_t solo_field = avro_string("Example field value");

avro_record_set(example, "solo", solo_field);

...
avro_datum_decref(example);
```
In this example, the solo_field datum would not be freed since it has two references: the original reference and a reference inside the Example record. The avro_datum_decref(example) call drops the number of reference to one. If you are finished with the solo_field schema, then you need to avro_schema_decref(solo_field) to completely dereference the solo_field datum and free it.

## 6. Wrap It and Give It
You’ll notice that some datatypes can be "wrapped" and "given". This allows C programmers the freedom to decide who is responsible for the memory. Let’s take strings for example.

To create a string datum, you have three different methods:
```c
avro_datum_t avro_string(const char *str);
avro_datum_t avro_wrapstring(const char *str);
avro_datum_t avro_givestring(const char *str);
```
If you use, avro_string then Avro C will make a copy of your string and free it when the datum is dereferenced. In some cases, especially when dealing with large amounts of data, you want to avoid this memory copy. That’s where avro_wrapstring and avro_givestring can help.

If you use, avro_wrapstring then Avro C will do no memory management at all. It will just save a pointer to your data and it’s your responsibility to free the string.

Warning
When using avro_wrapstring, do not free the string before you dereference the string datum with avro_datum_decref().
Lastly, if you use avro_givestring then Avro C will free the string later when the datum is dereferenced. In a sense, you are "giving" responsibility for freeing the string to Avro C.

Warning
Don’t "give" Avro C a string that you haven’t allocated from the heap with e.g. malloc or strdup.

For example, don’t do this:

avro_datum_t bad_idea = avro_givestring("This isn't allocated on the heap");
## 7. Schema Validation
If you want to write a datum, you would use the following function
```c
int avro_write_data(avro_writer_t writer,
                    avro_schema_t writers_schema, avro_datum_t datum);
```
If you pass in a writers_schema, then you datum will be validated before it is sent to the writer. This check ensures that your data has the correct format. If you are certain your datum is correct, you can pass a NULL value for writers_schema and Avro C will not validate before writing.

Note
Data written to an Avro File Object Container is always validated.
## 8. Examples
I’m not even supposed to be here today!

— Dante Hicks
Imagine you’re a free-lance hacker in Leonardo, New Jersey and you’ve been approached by the owner of the local Quick Stop Convenience store. He wants you to create a contact database case he needs to call employees to work on their day off.

You might build a simple contact system using Avro C like the following…
```c
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to you under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 * https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
 * implied.  See the License for the specific language governing
 * permissions and limitations under the License.
 */

#include <avro.h>
#include <stdio.h>
#include <stdlib.h>

#ifdef DEFLATE_CODEC
#define QUICKSTOP_CODEC  "deflate"
#else
#define QUICKSTOP_CODEC  "null"
#endif

avro_schema_t person_schema;
int64_t id = 0;

/* A simple schema for our tutorial */
const char  PERSON_SCHEMA[] =
"{\"type\":\"record\",\
  \"name\":\"Person\",\
  \"fields\":[\
     {\"name\": \"ID\", \"type\": \"long\"},\
     {\"name\": \"First\", \"type\": \"string\"},\
     {\"name\": \"Last\", \"type\": \"string\"},\
     {\"name\": \"Phone\", \"type\": \"string\"},\
     {\"name\": \"Age\", \"type\": \"int\"}]}";

/* Parse schema into a schema data structure */
void init_schema(void)
{
        if (avro_schema_from_json_literal(PERSON_SCHEMA, &person_schema)) {
                fprintf(stderr, "Unable to parse person schema\n");
                exit(EXIT_FAILURE);
        }
}

/* Create a value to match the person schema and save it */
void
add_person(avro_file_writer_t db, const char *first, const char *last,
           const char *phone, int32_t age)
{
        avro_value_iface_t  *person_class =
            avro_generic_class_from_schema(person_schema);

        avro_value_t  person;
        avro_generic_value_new(person_class, &person);

        avro_value_t id_value;
        avro_value_t first_value;
        avro_value_t last_value;
        avro_value_t age_value;
        avro_value_t phone_value;

        if (avro_value_get_by_name(&person, "ID", &id_value, NULL) == 0) {
                avro_value_set_long(&id_value, ++id);
        }
        if (avro_value_get_by_name(&person, "First", &first_value, NULL) == 0) {
                avro_value_set_string(&first_value, first);
        }
        if (avro_value_get_by_name(&person, "Last", &last_value, NULL) == 0) {
                avro_value_set_string(&last_value, last);
        }
        if (avro_value_get_by_name(&person, "Age", &age_value, NULL) == 0) {
                avro_value_set_int(&age_value, age);
        }
        if (avro_value_get_by_name(&person, "Phone", &phone_value, NULL) == 0) {
                avro_value_set_string(&phone_value, phone);
        }

        if (avro_file_writer_append_value(db, &person)) {
                fprintf(stderr,
                        "Unable to write Person value to memory buffer\nMessage: %s\n", avro_strerror());
                exit(EXIT_FAILURE);
        }

        /* Decrement all our references to prevent memory from leaking */
        avro_value_decref(&person);
        avro_value_iface_decref(person_class);
}

int print_person(avro_file_reader_t db, avro_schema_t reader_schema)
{

        avro_value_iface_t  *person_class =
            avro_generic_class_from_schema(person_schema);

        avro_value_t person;
        avro_generic_value_new(person_class, &person);

        int rval;

        rval = avro_file_reader_read_value(db, &person);
        if (rval == 0) {
                int64_t id;
                int32_t age;
                int32_t *p;
                size_t size;
                avro_value_t id_value;
                avro_value_t first_value;
                avro_value_t last_value;
                avro_value_t age_value;
                avro_value_t phone_value;

                if (avro_value_get_by_name(&person, "ID", &id_value, NULL) == 0) {
                        avro_value_get_long(&id_value, &id);
                        fprintf(stdout, "%"PRId64" | ", id);
                }
                if (avro_value_get_by_name(&person, "First", &first_value, NULL) == 0) {
                        avro_value_get_string(&first_value, &p, &size);
                        fprintf(stdout, "%15s | ", p);
                }
                if (avro_value_get_by_name(&person, "Last", &last_value, NULL) == 0) {
                        avro_value_get_string(&last_value, &p, &size);
                        fprintf(stdout, "%15s | ", p);
                }
                if (avro_value_get_by_name(&person, "Phone", &phone_value, NULL) == 0) {
                        avro_value_get_string(&phone_value, &p, &size);
                        fprintf(stdout, "%15s | ", p);
                }
                if (avro_value_get_by_name(&person, "Age", &age_value, NULL) == 0) {
                        avro_value_get_int(&age_value, &age);
                        fprintf(stdout, "%"PRId32" | ", age);
                }
                fprintf(stdout, "\n");

                /* We no longer need this memory */
                avro_value_decref(&person);
                avro_value_iface_decref(person_class);
        }
        return rval;
}

int main(void)
{
        int rval;
        avro_file_reader_t dbreader;
        avro_file_writer_t db;
        avro_schema_t projection_schema, first_name_schema, phone_schema;
        int64_t i;
        const char *dbname = "quickstop.db";
        char number[15] = {0};

        /* Initialize the schema structure from JSON */
        init_schema();

        /* Delete the database if it exists */
        remove(dbname);
        /* Create a new database */
        rval = avro_file_writer_create_with_codec
            (dbname, person_schema, &db, QUICKSTOP_CODEC, 0);
        if (rval) {
                fprintf(stderr, "There was an error creating %s\n", dbname);
                fprintf(stderr, " error message: %s\n", avro_strerror());
                exit(EXIT_FAILURE);
        }

        /* Add lots of people to the database */
        for (i = 0; i < 1000; i++)
        {
                sprintf(number, "(%d)", (int)i);
                add_person(db, "Dante", "Hicks", number, 32);
                add_person(db, "Randal", "Graves", "(555) 123-5678", 30);
                add_person(db, "Veronica", "Loughran", "(555) 123-0987", 28);
                add_person(db, "Caitlin", "Bree", "(555) 123-2323", 27);
                add_person(db, "Bob", "Silent", "(555) 123-6422", 29);
                add_person(db, "Jay", "???", number, 26);
        }

        /* Close the block and open a new one */
        avro_file_writer_flush(db);
        add_person(db, "Super", "Man", "123456", 31);

        avro_file_writer_close(db);

        fprintf(stdout, "\nNow let's read all the records back out\n");

        /* Read all the records and print them */
        if (avro_file_reader(dbname, &dbreader)) {
                fprintf(stderr, "Error opening file: %s\n", avro_strerror());
                exit(EXIT_FAILURE);
        }
        for (i = 0; i < id; i++) {
                if (print_person(dbreader, NULL)) {
                        fprintf(stderr, "Error printing person\nMessage: %s\n", avro_strerror());
                        exit(EXIT_FAILURE);
                }
        }
        avro_file_reader_close(dbreader);

        /* You can also use projection, to only decode only the data you are
           interested in.  This is particularly useful when you have
           huge data sets and you'll only interest in particular fields
           e.g. your contacts First name and phone number */
        projection_schema = avro_schema_record("Person", NULL);
        first_name_schema = avro_schema_string();
        phone_schema = avro_schema_string();
        avro_schema_record_field_append(projection_schema, "First",
                                        first_name_schema);
        avro_schema_record_field_append(projection_schema, "Phone",
                                        phone_schema);

        /* Read only the record you're interested in */
        fprintf(stdout,
                "\n\nUse projection to print only the First name and phone numbers\n");
        if (avro_file_reader(dbname, &dbreader)) {
                fprintf(stderr, "Error opening file: %s\n", avro_strerror());
                exit(EXIT_FAILURE);
        }
        for (i = 0; i < id; i++) {
                if (print_person(dbreader, projection_schema)) {
                        fprintf(stderr, "Error printing person: %s\n",
                                avro_strerror());
                        exit(EXIT_FAILURE);
                }
        }
        avro_file_reader_close(dbreader);
        avro_schema_decref(first_name_schema);
        avro_schema_decref(phone_schema);
        avro_schema_decref(projection_schema);

        /* We don't need this schema anymore */
        avro_schema_decref(person_schema);
        return 0;
}
```
When you compile and run this program, you should get the following output
```
Successfully added Hicks, Dante id=1
Successfully added Graves, Randal id=2
Successfully added Loughran, Veronica id=3
Successfully added Bree, Caitlin id=4
Successfully added Silent, Bob id=5
Successfully added ???, Jay id=6
```
Avro is compact. Here is the data for all 6 people.
```
| 02 0A 44 61 6E 74 65 0A | 48 69 63 6B 73 1C 28 35 |   ..Dante.Hicks.(5
| 35 35 29 20 31 32 33 2D | 34 35 36 37 40 04 0C 52 |   55) 123-4567@..R
| 61 6E 64 61 6C 0C 47 72 | 61 76 65 73 1C 28 35 35 |   andal.Graves.(55
| 35 29 20 31 32 33 2D 35 | 36 37 38 3C 06 10 56 65 |   5) 123-5678<..Ve
| 72 6F 6E 69 63 61 10 4C | 6F 75 67 68 72 61 6E 1C |   ronica.Loughran.
| 28 35 35 35 29 20 31 32 | 33 2D 30 39 38 37 38 08 |   (555) 123-09878.
| 0E 43 61 69 74 6C 69 6E | 08 42 72 65 65 1C 28 35 |   .Caitlin.Bree.(5
| 35 35 29 20 31 32 33 2D | 32 33 32 33 36 0A 06 42 |   55) 123-23236..B
| 6F 62 0C 53 69 6C 65 6E | 74 1C 28 35 35 35 29 20 |   ob.Silent.(555)
| 31 32 33 2D 36 34 32 32 | 3A 0C 06 4A 61 79 06 3F |   123-6422:..Jay.?
| 3F 3F 1C 28 35 35 35 29 | 20 31 32 33 2D 39 31 38 |   ??.(555) 123-918
| 32 34 .. .. .. .. .. .. | .. .. .. .. .. .. .. .. |   24..............
```
Now let's read all the records back out
1 |           Dante |           Hicks |  (555) 123-4567 | 32
2 |          Randal |          Graves |  (555) 123-5678 | 30
3 |        Veronica |        Loughran |  (555) 123-0987 | 28
4 |         Caitlin |            Bree |  (555) 123-2323 | 27
5 |             Bob |          Silent |  (555) 123-6422 | 29
6 |             Jay |             ??? |  (555) 123-9182 | 26


Use projection to print only the First name and phone numbers
          Dante |  (555) 123-4567 |
         Randal |  (555) 123-5678 |
       Veronica |  (555) 123-0987 |
        Caitlin |  (555) 123-2323 |
            Bob |  (555) 123-6422 |
            Jay |  (555) 123-9182 |
The Quick Stop owner was so pleased, he asked you to create a movie database for his RST Video store.

## 9. Reference files
### 9.1. avro.h
The avro.h header file contains the complete public API for Avro C. The documentation is rather sparse right now but we’ll be adding more information soon.
```c
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to you under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 * https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
 * implied.  See the License for the specific language governing
 * permissions and limitations under the License.
 */
#ifndef AVRO_H
#define AVRO_H
#ifdef __cplusplus
extern "C" {
#define CLOSE_EXTERN }
#else
#define CLOSE_EXTERN
#endif

#include <avro/allocation.h>
#include <avro/basics.h>
#include <avro/consumer.h>
#include <avro/data.h>
#include <avro/errors.h>
#include <avro/generic.h>
#include <avro/io.h>
#include <avro/legacy.h>
#include <avro/platform.h>
#include <avro/resolver.h>
#include <avro/schema.h>
#include <avro/value.h>

CLOSE_EXTERN
#endif
```
### 9.2. test_avro_data.c
Another good way to learn how to encode/decode data in Avro C is to look at the test_avro_data.c unit test. This simple unit test checks that all the avro types can be encoded/decoded correctly.
```c
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to you under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 * https://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or
 * implied.  See the License for the specific language governing
 * permissions and limitations under the License.
 */

#include "avro.h"
#include "avro_private.h"
#include <limits.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

char buf[4096];
avro_reader_t reader;
avro_writer_t writer;

typedef int (*avro_test) (void);

/*
 * Use a custom allocator that verifies that the size that we use to
 * free an object matches the size that we use to allocate it.
 */

static void *
test_allocator(void *ud, void *ptr, size_t osize, size_t nsize)
{
        AVRO_UNUSED(ud);
        AVRO_UNUSED(osize);

        if (nsize == 0) {
                size_t  *size = ((size_t *) ptr) - 1;
                if (osize != *size) {
                        fprintf(stderr,
                                "Error freeing %p:\n"
                                "Size passed to avro_free (%" PRIsz ") "
                                "doesn't match size passed to "
                                "avro_malloc (%" PRIsz ")\n",
                                ptr, osize, *size);
                        abort();
                        //exit(EXIT_FAILURE);
                }
                free(size);
                return NULL;
        } else {
                size_t  real_size = nsize + sizeof(size_t);
                size_t  *old_size = ptr? ((size_t *) ptr)-1: NULL;
                size_t  *size = (size_t *) realloc(old_size, real_size);
                *size = nsize;
                return (size + 1);
        }
}

void init_rand(void)
{
        srand(time(NULL));
}

double rand_number(double from, double to)
{
        double range = to - from;
        return from + ((double)rand() / (RAND_MAX + 1.0)) * range;
}

int64_t rand_int64(void)
{
        return (int64_t) rand_number(LONG_MIN, LONG_MAX);
}

int32_t rand_int32(void)
{
        return (int32_t) rand_number(INT_MIN, INT_MAX);
}

void
write_read_check(avro_schema_t writers_schema, avro_datum_t datum,
                 avro_schema_t readers_schema, avro_datum_t expected, char *type)
{
        avro_datum_t datum_out;
        int validate;

        for (validate = 0; validate <= 1; validate++) {

                reader = avro_reader_memory(buf, sizeof(buf));
                writer = avro_writer_memory(buf, sizeof(buf));

                if (!expected) {
                        expected = datum;
                }

                /* Validating read/write */
                if (avro_write_data
                    (writer, validate ? writers_schema : NULL, datum)) {
                        fprintf(stderr, "Unable to write %s validate=%d\n  %s\n",
                                type, validate, avro_strerror());
                        exit(EXIT_FAILURE);
                }
                int64_t size =
                    avro_size_data(writer, validate ? writers_schema : NULL,
                                   datum);
                if (size != avro_writer_tell(writer)) {
                        fprintf(stderr,
                                "Unable to calculate size %s validate=%d "
                                "(%"PRId64" != %"PRId64")\n  %s\n",
                                type, validate, size, avro_writer_tell(writer),
                                avro_strerror());
                        exit(EXIT_FAILURE);
                }
                if (avro_read_data
                    (reader, writers_schema, readers_schema, &datum_out)) {
                        fprintf(stderr, "Unable to read %s validate=%d\n  %s\n",
                                type, validate, avro_strerror());
                        fprintf(stderr, "  %s\n", avro_strerror());
                        exit(EXIT_FAILURE);
                }
                if (!avro_datum_equal(expected, datum_out)) {
                        fprintf(stderr,
                                "Unable to encode/decode %s validate=%d\n  %s\n",
                                type, validate, avro_strerror());
                        exit(EXIT_FAILURE);
                }

                avro_reader_dump(reader, stderr);
                avro_datum_decref(datum_out);
                avro_reader_free(reader);
                avro_writer_free(writer);
        }
}

static void test_json(avro_datum_t datum, const char *expected)
{
        char  *json = NULL;
        avro_datum_to_json(datum, 1, &json);
        if (strcasecmp(json, expected) != 0) {
                fprintf(stderr, "Unexpected JSON encoding: %s\n", json);
                exit(EXIT_FAILURE);
        }
        free(json);
}

static int test_string(void)
{
        unsigned int i;
        const char *strings[] = { "Four score and seven years ago",
                "our father brought forth on this continent",
                "a new nation", "conceived in Liberty",
                "and dedicated to the proposition that all men are created equal."
        };
        avro_schema_t writer_schema = avro_schema_string();
        for (i = 0; i < sizeof(strings) / sizeof(strings[0]); i++) {
                avro_datum_t datum = avro_givestring(strings[i], NULL);
                write_read_check(writer_schema, datum, NULL, NULL, "string");
                avro_datum_decref(datum);
        }

        avro_datum_t  datum = avro_givestring(strings[0], NULL);
        test_json(datum, "\"Four score and seven years ago\"");
        avro_datum_decref(datum);

        // The following should bork if we don't copy the string value
        // correctly (since we'll try to free a static string).

        datum = avro_string("this should be copied");
        avro_string_set(datum, "also this");
        avro_datum_decref(datum);

        avro_schema_decref(writer_schema);
        return 0;
}

static int test_bytes(void)
{
        char bytes[] = { 0xDE, 0xAD, 0xBE, 0xEF };
        avro_schema_t writer_schema = avro_schema_bytes();
        avro_datum_t datum;
        avro_datum_t expected_datum;

        datum = avro_givebytes(bytes, sizeof(bytes), NULL);
        write_read_check(writer_schema, datum, NULL, NULL, "bytes");
        test_json(datum, "\"\\u00de\\u00ad\\u00be\\u00ef\"");
        avro_datum_decref(datum);
        avro_schema_decref(writer_schema);

        datum = avro_givebytes(NULL, 0, NULL);
        avro_givebytes_set(datum, bytes, sizeof(bytes), NULL);
        expected_datum = avro_givebytes(bytes, sizeof(bytes), NULL);
        if (!avro_datum_equal(datum, expected_datum)) {
                fprintf(stderr,
                        "Expected equal bytes instances.\n");
                exit(EXIT_FAILURE);
        }
        avro_datum_decref(datum);
        avro_datum_decref(expected_datum);

        // The following should bork if we don't copy the bytes value
        // correctly (since we'll try to free a static string).

        datum = avro_bytes("original", 8);
        avro_bytes_set(datum, "alsothis", 8);
        avro_datum_decref(datum);

        avro_schema_decref(writer_schema);
        return 0;
}

static int test_int32(void)
{
        int i;
        avro_schema_t writer_schema = avro_schema_int();
        avro_schema_t long_schema = avro_schema_long();
        avro_schema_t float_schema = avro_schema_float();
        avro_schema_t double_schema = avro_schema_double();
        for (i = 0; i < 100; i++) {
                int32_t  value = rand_int32();
                avro_datum_t datum = avro_int32(value);
                avro_datum_t long_datum = avro_int64(value);
                avro_datum_t float_datum = avro_float(value);
                avro_datum_t double_datum = avro_double(value);
                write_read_check(writer_schema, datum, NULL, NULL, "int");
                write_read_check(writer_schema, datum,
                                 long_schema, long_datum, "int->long");
                write_read_check(writer_schema, datum,
                                 float_schema, float_datum, "int->float");
                write_read_check(writer_schema, datum,
                                 double_schema, double_datum, "int->double");
                avro_datum_decref(datum);
                avro_datum_decref(long_datum);
                avro_datum_decref(float_datum);
                avro_datum_decref(double_datum);
        }

        avro_datum_t  datum = avro_int32(10000);
        test_json(datum, "10000");
        avro_datum_decref(datum);

        avro_schema_decref(writer_schema);
        avro_schema_decref(long_schema);
        avro_schema_decref(float_schema);
        avro_schema_decref(double_schema);
        return 0;
}

static int test_int64(void)
{
        int i;
        avro_schema_t writer_schema = avro_schema_long();
        avro_schema_t float_schema = avro_schema_float();
        avro_schema_t double_schema = avro_schema_double();
        for (i = 0; i < 100; i++) {
                int64_t  value = rand_int64();
                avro_datum_t datum = avro_int64(value);
                avro_datum_t float_datum = avro_float(value);
                avro_datum_t double_datum = avro_double(value);
                write_read_check(writer_schema, datum, NULL, NULL, "long");
                write_read_check(writer_schema, datum,
                                 float_schema, float_datum, "long->float");
                write_read_check(writer_schema, datum,
                                 double_schema, double_datum, "long->double");
                avro_datum_decref(datum);
                avro_datum_decref(float_datum);
                avro_datum_decref(double_datum);
        }

        avro_datum_t  datum = avro_int64(10000);
        test_json(datum, "10000");
        avro_datum_decref(datum);

        avro_schema_decref(writer_schema);
        avro_schema_decref(float_schema);
        avro_schema_decref(double_schema);
        return 0;
}

static int test_double(void)
{
        int i;
        avro_schema_t schema = avro_schema_double();
        for (i = 0; i < 100; i++) {
                avro_datum_t datum = avro_double(rand_number(-1.0E10, 1.0E10));
                write_read_check(schema, datum, NULL, NULL, "double");
                avro_datum_decref(datum);
        }

        avro_datum_t  datum = avro_double(2000.0);
        test_json(datum, "2000.0");
        avro_datum_decref(datum);

        avro_schema_decref(schema);
        return 0;
}

static int test_float(void)
{
        int i;
        avro_schema_t schema = avro_schema_float();
        avro_schema_t double_schema = avro_schema_double();
        for (i = 0; i < 100; i++) {
                float  value = rand_number(-1.0E10, 1.0E10);
                avro_datum_t datum = avro_float(value);
                avro_datum_t double_datum = avro_double(value);
                write_read_check(schema, datum, NULL, NULL, "float");
                write_read_check(schema, datum,
                                 double_schema, double_datum, "float->double");
                avro_datum_decref(datum);
                avro_datum_decref(double_datum);
        }

        avro_datum_t  datum = avro_float(2000.0);
        test_json(datum, "2000.0");
        avro_datum_decref(datum);

        avro_schema_decref(schema);
        avro_schema_decref(double_schema);
        return 0;
}

static int test_boolean(void)
{
        int i;
        const char  *expected_json[] = { "false", "true" };
        avro_schema_t schema = avro_schema_boolean();
        for (i = 0; i <= 1; i++) {
                avro_datum_t datum = avro_boolean(i);
                write_read_check(schema, datum, NULL, NULL, "boolean");
                test_json(datum, expected_json[i]);
                avro_datum_decref(datum);
        }
        avro_schema_decref(schema);
        return 0;
}

static int test_null(void)
{
        avro_schema_t schema = avro_schema_null();
        avro_datum_t datum = avro_null();
        write_read_check(schema, datum, NULL, NULL, "null");
        test_json(datum, "null");
        avro_datum_decref(datum);
        return 0;
}

static int test_record(void)
{
        avro_schema_t schema = avro_schema_record("person", NULL);
        avro_schema_record_field_append(schema, "name", avro_schema_string());
        avro_schema_record_field_append(schema, "age", avro_schema_int());

        avro_datum_t datum = avro_record(schema);
        avro_datum_t name_datum, age_datum;

        name_datum = avro_givestring("Joseph Campbell", NULL);
        age_datum = avro_int32(83);

        avro_record_set(datum, "name", name_datum);
        avro_record_set(datum, "age", age_datum);

        write_read_check(schema, datum, NULL, NULL, "record");
        test_json(datum, "{\"name\": \"Joseph Campbell\", \"age\": 83}");

        int  rc;
        avro_record_set_field_value(rc, datum, int32, "age", 104);

        int32_t  age = 0;
        avro_record_get_field_value(rc, datum, int32, "age", &age);
        if (age != 104) {
                fprintf(stderr, "Incorrect age value\n");
                exit(EXIT_FAILURE);
        }

        avro_datum_decref(name_datum);
        avro_datum_decref(age_datum);
        avro_datum_decref(datum);
        avro_schema_decref(schema);
        return 0;
}

static int test_nested_record(void)
{
        const char  *json =
                "{"
                "  \"type\": \"record\","
                "  \"name\": \"list\","
                "  \"fields\": ["
                "    { \"name\": \"x\", \"type\": \"int\" },"
                "    { \"name\": \"y\", \"type\": \"int\" },"
                "    { \"name\": \"next\", \"type\": [\"null\",\"list\"]}"
                "  ]"
                "}";

        int  rval;

        avro_schema_t schema = NULL;
        avro_schema_error_t error;
        avro_schema_from_json(json, strlen(json), &schema, &error);

        avro_datum_t  head = avro_datum_from_schema(schema);
        avro_record_set_field_value(rval, head, int32, "x", 10);
        avro_record_set_field_value(rval, head, int32, "y", 10);

        avro_datum_t  next = NULL;
        avro_datum_t  tail = NULL;

        avro_record_get(head, "next", &next);
        avro_union_set_discriminant(next, 1, &tail);
        avro_record_set_field_value(rval, tail, int32, "x", 20);
        avro_record_set_field_value(rval, tail, int32, "y", 20);

        avro_record_get(tail, "next", &next);
        avro_union_set_discriminant(next, 0, NULL);

        write_read_check(schema, head, NULL, NULL, "nested record");

        avro_schema_decref(schema);
        avro_datum_decref(head);

        return 0;
}

static int test_enum(void)
{
        enum avro_languages {
                AVRO_C,
                AVRO_CPP,
                AVRO_PYTHON,
                AVRO_RUBY,
                AVRO_JAVA
        };
        avro_schema_t schema = avro_schema_enum("language");
        avro_datum_t datum = avro_enum(schema, AVRO_C);

        avro_schema_enum_symbol_append(schema, "C");
        avro_schema_enum_symbol_append(schema, "C++");
        avro_schema_enum_symbol_append(schema, "Python");
        avro_schema_enum_symbol_append(schema, "Ruby");
        avro_schema_enum_symbol_append(schema, "Java");

        if (avro_enum_get(datum) != AVRO_C) {
                fprintf(stderr, "Unexpected enum value AVRO_C\n");
                exit(EXIT_FAILURE);
        }

        if (strcmp(avro_enum_get_name(datum), "C") != 0) {
                fprintf(stderr, "Unexpected enum value name C\n");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, datum, NULL, NULL, "enum");
        test_json(datum, "\"C\"");

        avro_enum_set(datum, AVRO_CPP);
        if (strcmp(avro_enum_get_name(datum), "C++") != 0) {
                fprintf(stderr, "Unexpected enum value name C++\n");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, datum, NULL, NULL, "enum");
        test_json(datum, "\"C++\"");

        avro_enum_set_name(datum, "Python");
        if (avro_enum_get(datum) != AVRO_PYTHON) {
                fprintf(stderr, "Unexpected enum value AVRO_PYTHON\n");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, datum, NULL, NULL, "enum");
        test_json(datum, "\"Python\"");

        avro_datum_decref(datum);
        avro_schema_decref(schema);
        return 0;
}

static int test_array(void)
{
        int i, rval;
        avro_schema_t schema = avro_schema_array(avro_schema_int());
        avro_datum_t datum = avro_array(schema);

        for (i = 0; i < 10; i++) {
                avro_datum_t i32_datum = avro_int32(i);
                rval = avro_array_append_datum(datum, i32_datum);
                avro_datum_decref(i32_datum);
                if (rval) {
                        exit(EXIT_FAILURE);
                }
        }

        if (avro_array_size(datum) != 10) {
                fprintf(stderr, "Unexpected array size");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, datum, NULL, NULL, "array");
        test_json(datum, "[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]");
        avro_datum_decref(datum);
        avro_schema_decref(schema);
        return 0;
}

static int test_map(void)
{
        avro_schema_t schema = avro_schema_map(avro_schema_long());
        avro_datum_t datum = avro_map(schema);
        int64_t i = 0;
        char *nums[] =
            { "zero", "one", "two", "three", "four", "five", "six", NULL };
        while (nums[i]) {
                avro_datum_t i_datum = avro_int64(i);
                avro_map_set(datum, nums[i], i_datum);
                avro_datum_decref(i_datum);
                i++;
        }

        if (avro_array_size(datum) != 7) {
                fprintf(stderr, "Unexpected map size\n");
                exit(EXIT_FAILURE);
        }

        avro_datum_t value;
        const char  *key;
        avro_map_get_key(datum, 2, &key);
        avro_map_get(datum, key, &value);
        int64_t  val;
        avro_int64_get(value, &val);

        if (val != 2) {
                fprintf(stderr, "Unexpected map value 2\n");
                exit(EXIT_FAILURE);
        }

        int  index;
        if (avro_map_get_index(datum, "two", &index)) {
                fprintf(stderr, "Can't get index for key \"two\": %s\n",
                        avro_strerror());
                exit(EXIT_FAILURE);
        }
        if (index != 2) {
                fprintf(stderr, "Unexpected index for key \"two\"\n");
                exit(EXIT_FAILURE);
        }
        if (!avro_map_get_index(datum, "foobar", &index)) {
                fprintf(stderr, "Unexpected index for key \"foobar\"\n");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, datum, NULL, NULL, "map");
        test_json(datum,
                  "{\"zero\": 0, \"one\": 1, \"two\": 2, \"three\": 3, "
                  "\"four\": 4, \"five\": 5, \"six\": 6}");
        avro_datum_decref(datum);
        avro_schema_decref(schema);
        return 0;
}

static int test_union(void)
{
        avro_schema_t schema = avro_schema_union();
        avro_datum_t union_datum;
        avro_datum_t datum;
        avro_datum_t union_datum1;
        avro_datum_t datum1;

        avro_schema_union_append(schema, avro_schema_string());
        avro_schema_union_append(schema, avro_schema_int());
        avro_schema_union_append(schema, avro_schema_null());

        datum = avro_givestring("Follow your bliss.", NULL);
        union_datum = avro_union(schema, 0, datum);

        if (avro_union_discriminant(union_datum) != 0) {
                fprintf(stderr, "Unexpected union discriminant\n");
                exit(EXIT_FAILURE);
        }

        if (avro_union_current_branch(union_datum) != datum) {
                fprintf(stderr, "Unexpected union branch datum\n");
                exit(EXIT_FAILURE);
        }

        union_datum1 = avro_datum_from_schema(schema);
        avro_union_set_discriminant(union_datum1, 0, &datum1);
        avro_givestring_set(datum1, "Follow your bliss.", NULL);

        if (!avro_datum_equal(datum, datum1)) {
                fprintf(stderr, "Union values should be equal\n");
                exit(EXIT_FAILURE);
        }

        write_read_check(schema, union_datum, NULL, NULL, "union");
        test_json(union_datum, "{\"string\": \"Follow your bliss.\"}");

        avro_datum_decref(datum);
        avro_union_set_discriminant(union_datum, 2, &datum);
        test_json(union_datum, "null");

        avro_datum_decref(union_datum);
        avro_datum_decref(datum);
        avro_datum_decref(union_datum1);
        avro_schema_decref(schema);
        return 0;
}

static int test_fixed(void)
{
        char bytes[] = { 0xD, 0xA, 0xD, 0xA, 0xB, 0xA, 0xB, 0xA };
        avro_schema_t schema = avro_schema_fixed("msg", sizeof(bytes));
        avro_datum_t datum;
        avro_datum_t expected_datum;

        datum = avro_givefixed(schema, bytes, sizeof(bytes), NULL);
        write_read_check(schema, datum, NULL, NULL, "fixed");
        test_json(datum, "\"\\r\\n\\r\\n\\u000b\\n\\u000b\\n\"");
        avro_datum_decref(datum);

        datum = avro_givefixed(schema, NULL, sizeof(bytes), NULL);
        avro_givefixed_set(datum, bytes, sizeof(bytes), NULL);
        expected_datum = avro_givefixed(schema, bytes, sizeof(bytes), NULL);
        if (!avro_datum_equal(datum, expected_datum)) {
                fprintf(stderr,
                        "Expected equal fixed instances.\n");
                exit(EXIT_FAILURE);
        }
        avro_datum_decref(datum);
        avro_datum_decref(expected_datum);

        // The following should bork if we don't copy the fixed value
        // correctly (since we'll try to free a static string).

        datum = avro_fixed(schema, "original", 8);
        avro_fixed_set(datum, "alsothis", 8);
        avro_datum_decref(datum);

        avro_schema_decref(schema);
        return 0;
}

int main(void)
{
        avro_set_allocator(test_allocator, NULL);

        unsigned int i;
        struct avro_tests {
                char *name;
                avro_test func;
        } tests[] = {
                {
                "string", test_string}, {
                "bytes", test_bytes}, {
                "int", test_int32}, {
                "long", test_int64}, {
                "float", test_float}, {
                "double", test_double}, {
                "boolean", test_boolean}, {
                "null", test_null}, {
                "record", test_record}, {
                "nested_record", test_nested_record}, {
                "enum", test_enum}, {
                "array", test_array}, {
                "map", test_map}, {
                "fixed", test_fixed}, {
                "union", test_union}
        };

        init_rand();
        for (i = 0; i < sizeof(tests) / sizeof(tests[0]); i++) {
                struct avro_tests *test = tests + i;
                fprintf(stderr, "**** Running %s tests ****\n", test->name);
                if (test->func() != 0) {
                        return EXIT_FAILURE;
                }
        }
        return EXIT_SUCCESS;
}
```