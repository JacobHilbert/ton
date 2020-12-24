# TON: Tab Object Notation

Dictionary serialization format that uses indentation (`\t`) to make the structure human readable. Syntax is python-inspired, without commas or brackets. For instance,
```python
"name": "Jane Doe"
"age": 20
"grades": 
	"homeworks":75.81
	"assistance":59.00
	"exams": 85.14  49.222  93.7
"friends":
		"Alex"
		"Sora"
		"Kathy"
		"Jay"
		
		"Valentine"
		"Karla"
		"Rowan"
		"Max"
```
Would be equivalent to Python's
```python
{
"name": "Jane Doe",
"age": 20,
"grades": {
	"homeworks":75.81,
	"assistance":59.00,
	"exams": [85.14, 49.222, 93.7]
}
"friends": [
	[
		"Alex",
		"Sora",
		"Kathy",
		"Jay"
	],[	
		"Valentine",
		"Karla",
		"Rowan",
		"Max"
	]
]
}
```
(Which is valid in both Python and JSON)



## Installation

Clone this repo and

```
$ pip install .
```

# What is allowed

## Hashable types (key-valid)

### Numbers

* Arbitrary size integers
	* Octal and hexadecimal inputs are allowed, but it will not be persistent on load/dump. Examples:
		* `0o4324` (2260)
		* `0x4324` (17188)
* Floats and complex
	* Zero's sign may not be persistent on load/dump.
	* IEEE-754 special values can be entered directly as `inf` and `nan`, possibly signed, without quotes. These are case insensitive, so `nan` and `Nan` are the same thing.
	* Imaginary unit can be expressed as `i`, `im` or `j`, case insensitive, and with no multiplication sign on the complex part. Examples:
		* `1+3j`
		* `-10im`
		* `inf-nanI`
	* Precision could be lost if data is loaded and then dumped on a 32-bit machine.


### Strings

* Single `'` and double `"` quoted for single line strings are valid and the same. Both `'''` and `"""` are used for multi line.
* You can always write escaped quotes `\"`. Quotes different than the current string need no escaping: `" ' "`, `''' ' '''`, `""" ' """`.
* If string keys are valid python identifiers, they need not to be quoted:
```python
this_is_valid: "but this need to be quoted"
ia48_8a_á: "Python's language reference have the specific unicode groups that are valid here"
```
* First newline on multiline string is removed.Indentation of the closing triple-quote is not important. The following are the same.
```python
example_1: '''hello'''
example_2: '''
	hello
'''
example_3: '''
	hello
	'''
```

### Boolean and None/Null

* Booleans are case insensitive.
* `None` and `Null` are both valid and the same.

## Collections (unhashable, not valid as keys)

* Dictionaries can be arbitrarily nested; values can be anything. Dicts and lists cannot be keys, as they are not hashable.
* Lists can contain any combination of types, and can be also arbitrary nested.
* You can write one line structures, if you want. It should be bracketed and comma separated. Examples:
	* `{"a":3,"b:4"}`
	* `[1,2,3,4]`
* Empty dicts and lists are `{}` and `[]`

See Structuring for a better description of how to write nested structures. Or you could construct a `test` object that mimics the structure you want, and run:
```python
>>> print(ton.dumps(test))
```
To discover it yourself.

### Comments

Same as Python's: a `#` outside a string turns the rest of the line into a comment. Comments are ignored during load/dump.

## What is not allowed

* Functions, function calls or structure indexation (`a[0]`)
* Sets, frozensets, tuples, queues, `numpy.array`, etc. 
	* You can save them as lists and cast them after loading. But maybe there is a case for tuples and frozenmaps.
* String interpolation.
* Mathematical operations. The only allowed use of `+` and `-` are as part of complex numbers syntax.
* `Decimal` or `Fraction` numbers. 
	* You can save decimals and fractions as strings, and cast after loading. `Decimal('137.035999206')` and `Fraction('1/137')` are a thing.

# Structuring

> (again, you can create a `test` dict with the structure you want, and run `print(ton.dumps(test))` to see how to write it)

The idea is that a human reading the data would be able to infer the type of brackets of the structure. If you se a `key:value` pair, is a `dict`, if you see just a bunch of values, is a `list`. 

Additionaly, to stay sane, each indentation level means that the structure is nested one time; for instance, three indentation levels means that the element is contained in three structures (like a file in a folder in a folder in a folder).

```python
#{
"key":"value"
"list": #[
	"value 1"
	"value 2"
	"value 3"
#]
"dict": #{
	"key 1" : "value 1"
	"key 2" : "value 2"
	"key 3" : "value 3"
#}
#}
```

But what about, say, a list of lists? Well, that would be two levels of indentation. In Python/JSON it would be

```python
"list of lists":[
	[
		"a",
		"b",
		"c"
	],[
		"A",
		"B",
		"C"
	]
]
```

If we simply remove the commas and the brackets, we get

```python
"list of lists":
		"a"
		"b"
		"c"
		"A"
		"B"
		"C"
```

So we cannot longer tell where the first list ends and the second begins. Therefore our third structure rule is to separate structures at the same indentation level with a blank line, like this:


```python
"list of lists":# [ [
		"a"
		"b"
		"c"
	#],[
		"A"
		"B"
		"C"
# ] ]
```

### Beware, Single element lists

This is a dict from letters to numbers
```python
"a":1
"b":2
"c":3
```
And would be parsed as `{"a":1, "b":2, "c":3}`. This is a dict from letters to __lists__ of one element:
```python
"a":
	1
"b":
	2
"c":
	3
```


### Worked example

Using what we have, we can tell this is invalid:
```python
"Am I a list or a dict?":
	"a":1
	"b":2
	"c"
	"d"
```

Because it mixes `key:value` pairs and plain values. `"a":1, "b":2, "c", "d"` is neither a list nor a dict, so it makes no sense. But if we leave a blank line between the two
```python
"Am I a list or a dict?":
	"a":1
	"b":2

	"c"
	"d"
```
It becomes possible to infer the appropiate brackets. `"a":1, "b:2"` is a sequence of `key:value` pairs, so is a dict, and `"c", "d"` is a sequence of values, so is a list.
```python
"Am I a list or a dict?":
#{
	"a":1
	"b":2
#}
#[
	"c"
	"d"
#]
```
But the indentation is wrong, isn't it? Because one level of indentation sould mean that each the dict and the list must be contained in only one structure, but the key at the beginning should have only one value. If we put another indentation in both, it finally makes sense 
```python
"Am I a list or a dict?":
#[
	#{
		"a":1
		"b":2
	#}
	#[
		"c"
		"d"
	#]
#]
```
So, it is a list consisting of a dict and another list.


# To Do

I was thinking about allowing one line lists without brackets or commas (tab separated), so you could write something like
```python
"Toeplitz symmetric matrix":
	1	2	3	4
	2	1	2	3
	3	2	1	2
	4	3	2	1
```
And it would be parsed like
```python
{"Toeplitz symmetric matrix": [
	[1,	2,	3,	4],
	[2,	1,	2,	3],
	[3,	2,	1,	2],
	[4,	3,	2,	1]
]}
```

Or even high dimensional arrays, like
```python
"Levi-Civita symbol":
		0	0	0
		0	0	1
		0	-1	0
		
		0	0	1
		0	0	0
		1	0	0
		
		0	1	0
		-1	0	0
		0	0	0
