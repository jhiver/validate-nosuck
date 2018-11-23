# vac

vac is a library which validates and sanitizes your JSON inputs.

# sorry...

The samples below are written in coffeescript, cause it's what I like to write.
You don't need coffeescript to use this module though.


# the simple stuff


first say you have a structure like this:

    person =
      name: "John"
      surname: "Doe"
      age: 17
      foobar: 'baz'


You'd write a schema as follows:

    vac = require 'validate-and-clean'
    person_schema =
      _: vac().filter("name surname age".split(' '))
      name: vac().string().trim().minLen(3).maxLen(50)
      surname: vac().string().trim().minLen(3).maxLen(50)
      age: vac().optional().number().minVal(18)


Then you would validate and sanitize your object like so

    vac.validate(person).with(person_schema)
      .then (errors) ->
        if(errors)
          console.log "validation errors:", errors
        else
          console.log "there were no errors."

        console.log "person has been sanitized to:"
        console.log person

    person_schema =
    nosuck = require 'validate-nosuck'
    check = nosuck.check
    schema =
      name: check().string().minLen(3).maxLen(50)
      surname: check().string().minLen(3).maxLen(50)
      age: check().optional().number()


And you'd get the following output:

    validation errors: { age: 'minVal' }
    person is now: { name: 'John', surname: 'Doe', age: 17, foobar: 'baz' }


# built in validators / sanitizers

| check name      | description                                               |
|-----------------|-----------------------------------------------------------|
| default(val)    | sets a default value if undefined or null                 |
| email           | checks if a valid email                                   |
| equals(val)     | checks if equal to `val`                                  |
| notEquals       | checks if not equal to `val`                              |
| hasDigit        | checks if contains at least one digit                     |
| hasLowerCase    | checks if contains at least one lowercase letter          |
| hasSpecial      | checks if contains at least one special character         |
| hasUpperCase    | checks if contains at least one uppercase letter          |
| integer         | checks if is an integer value                             |
| like(regexp)    | checks if matches regexp                                  |
| unlike(regexp)  | checks unless matches regexp                              |
| maxLen(val)     | checks if has at most `val` elements                      |
| maxVal(val)     | checks if value is at most `val`                          |
| minLen(val)     | checks if has at least `val` elements                     |
| minVal(val)     | checks if value is at least `val`                         |
| number          | casts the value into a number unless it's NaN             |
| optional        | if undefined or null, this chain of checks will pass      |
| overwrite(val)  | overwrites with `val`                                     |
| pick            | restrict an object to a certain set of attributes         |
| round(decimals) | rounds a number to decimal places (default: 0)            |
| string          | casts the value into a string                             |
| trim            | trims a string                                            |
| uuid            | checks if looks like an uuid                              |
| isArray         | checks if it's an array                                   |
| schema          | these two are used for nested structures...               |
| each            | see examples below!                                       |

Is there an obvious one I missed? Let me know or better, do a pull request!
(on the index.coffee file please)

# add your own checks

Say an object has a user_id attribute, and you want to make sure it exists
replace the attribute with a `user` object:

    vac.register 'user', ->
      (attribute, object) ->
        id = object[attribute]
        if id is null then return Promise.resolve 'null'
        if id is undefined then return Promise.resolve 'undefined'
        return new Promise (resolve, reject) ->
          storage.getUserById id, (error, user) ->
            if error then return reject(error)
            unless user then return resolve('user') # throw error

            # it worked!
            delete object[attribute]
            object.user = user
            resolve()

You can now use this in your schema:

    schema.user_id = vac().user()


# things to remember

* Your check can alter the model / object which you validate!
  You should make a deep copy of your object if / when needed

* If your check resolves(), the check will pass and go to the
  next check in the chain

* If your check resolves(null), the check will pass unconditionnally
  (stop the chain for this attribute)

* If your check resolves anything else, it will be considered an error.

* It's using Promises throughout so we can transparently do some cool
  async stuff.


# Handling lists

Say we have a list of interests with a person:

    person =
      id: 12
      name: "John"
      surname: "Doe"
      age: 23
      likes: [ 'kiwis', 'bananas', 'beer' ]

And say we have written a 'isValidIngredient' check. You could do:

    schema =
      name: check().string().minLen(3).maxLen(50)
      surname: check().string().minLen(3).maxLen(50)
      age: check().optional().number()
      likes: check().isArray().each(
        check().isValidIngredient())
      )


# Handling nested objects

We can validate attributes which have an object as a value
against a new schema also.

    schema =
      name: check().string().minLen(3).maxLen(50)
      surname: check().string().minLen(3).maxLen(50)
      age: check().optional().number()
      details: check().schema(
        limbs: check().integer().minVal(0).maxVal(4)
        eyes: check().integer().minVal(0).maxVal(2)
        noses: check().integer().minVal(0).maxVal(1)
      )


# Lists can also contain schemas...

which can even reference themselves!

    personSchema =
      name: check().string().minLen(3).maxLen(50)
      surname: check().string().minLen(3).maxLen(50)
      age: check().optional().number()

    personSchema.friends = check().optional().isArray().each(
      check().schema(personSchema)
    )


# Error output format for lists & nested objects

Say we have

    person =
      id: 12
      name: "John"
      surname: "Doe"
      age: -1
      likes: [ 'kiwis', 'bananas', 'arsenic' ]
      details:
        limbs: 5
        eyes: 2
        noses: 1

After validation, object could look like:

    {
      "age": "minVal",
      "likes": [
        null,
        null,
        "badForYou"
      ],
      "details": {
        "limbs": "maxVal"
      }
    }


EXPORTS
-------

- vac()
- vac.register()
- vac.validate()


BUGS
----

If you find any, please drop me an email - jhiver (at) gmail (dot) com.
Patches & pull requests are always welcome of course!


ALSO
-----

This module free software and is distributed under the same license as node.js
itself (MIT license)

Copyright (C) 2018 - Jean-Michel Hiver

Permission is hereby granted, free of charge, to any person obtaining a copy of
this software and associated documentation files (the "Software"), to deal in
the Software without restriction, including without limitation the rights to
use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies
of the Software, and to permit persons to whom the Software is furnished to do
so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.