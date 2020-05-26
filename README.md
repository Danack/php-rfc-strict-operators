# PHP RFC: Strict operators directive

  - Version: 1.2
  - Date: 2020-04-09 (first version: 2019-05-25)
  - Author: Arnold Daniels, jasny@php.net
  - Status: Under Discussion
  - First Published at: <http://wiki.php.net/rfc/strict_operators>

## Introduction

PHP performs implicit type conversion for most operators. The rules of
conversion are complex, depending on the operator as well as on the type
and value of the operands. This can lead to surprising results, where a
statement seemingly contradicts itself. This RFC proposes a new
directive `strict_operators`, which limits the type juggling done by
operators and makes them throw a `TypeError` for unsupported types.

Making significant changes to the behavior of operators has significant
consequences to backward compatibility. Additionally, there is a
significant group of people who are in favor of the current method of
type juggling. Following the rationale of [PHP RFC: Scalar Type
Declarations](/rfc/scalar_type_hints_v5); an optional directive ensures
backward compatibility and allows people to choose the type checking
model that suits them best.

## Motivating examples

### Mixed type comparison

Mathematics states that "if `(a > b)` and `(b > c)`, then `(a > c)`".
This statement can be asserted in PHP;

``` php
if (($a > $b) && ($b > $c)) {
    assert($a > $c);
}
```

This assertion fails when choosing values of different types

``` php
$a = '42';
$b = 10;
$c = '9 eur';

if (($a > $b) && ($b > $c)) {
    assert($a > $c);
}
```

### Numeric string comparison

Non-strict comparison uses a "smart" comparison method that treats
strings as numbers if they are numeric. The meaning of the operator
changes based on the value of both operands.

Using the `<=>` operator to order the values of an array can lead to
different results based on the initial state of the array.

``` php
function sorted(array $arr) {
  usort($arr, function($x, $y) { return $x <=> $y; });
  return $arr;
}

sorted(['100', '5 eur', '62']); // ['100', '5 eur', '62']
sorted(['100', '62', '5 eur']); // ['5 eur', '62', '100']
sorted(['62', '100', '5 eur']); // ['62', '100', '5 eur']
```

### Array comparison

Using the `>`, `>=`, `<`, `<=` and `<=>` operators on arrays or objects
that don't have the same keys in the same order gives unexpected
results.

In the following example `$a` is both greater than and less than `$b`

``` php
$a = ['x' => 1, 'y' => 22];
$b = ['y' => 10, 'x' => 15];

$a > $b; // true
$a < $b; // true
```

The logic of relational operators other than `==`, `===`, `!=` and `!==`
has limited practical use. In case both arrays have the same keys (in
the same order), a side-by-side comparison is done. If the size differs,
the array with the most elements is always seen as the greatest;

``` php
[1] < [50]; // true
[1, 1] < [50]; // false
```

This is not a proper method to compare the size of the array, as two
operands of equal size but different values are not equal. Instead,
`count()` should be used in this case.

If two arrays have the same number of items but not the same keys, the
`<`, `<=`, `>` and `>=` operators will always return false.

``` php
[1] < ['bar' => 50]; // false
[1] > ['bar' => 50]; // false
```

In case the two arrays have the same number of items and the same keys
but in a different order, an element by element comparison is done. The
`>` and `>=` operator is implemented as the inverse of `<` and `<=`.
This results in walking through the operand that's expected to be the
smallest.

``` php
$a = ['x' => 1, 'y' => 22];
$b = ['y' => 10, 'x' => 15];

$a > $b; // true
$a < $b; // true
```

In the statement with the `>` operator, we walk through the elements of
`$b`, so first comparing `$b['y']` to `$a['y']`. In the statement with
`<` we walk through the elements of `$a`, so first comparing `$a['x']`
to `$b['x']`. This results in both statements, while seemingly
contracting, to evaluate to true.

### Strict vs non-strict comparison of arrays

Strict comparison requires that arrays have keys occurring in the same
order, while non-strict comparison allows out-of-order keys.

``` php
['a' => 'foo', 'b' => 'bar'] == ['b' => 'bar', 'a' => 0]; // true
```

To compare the values of two arrays in a strict way while not concerned
about the order, requires ordering the array by key prior to comparison.

### Switch control structure

The `switch` statement does a non-strict comparison. This can lead to
unexpected results;

``` php
function match($value)
{
  switch ($value) {
    case 2:
      return "double";
      break;
    case 1:
      return "single";
      break;
    case 0:
      return "none";
      break;
    default:
      throw new Exception("Unexpected value");
  }
}

match("foo"); // "none"
```

In case both the expression and the condition operands are both numeric
strings, both are converted to an integer. This can be unexpected;

``` php
function match($value)
{
  switch ($value) {
    case "1e1":
      return "1e1";
      break;
    case "10":
      return "10";
      break;
    default:
      throw new Exception("Unexpected value");
  }
}

match("10"); // "1e1"
```

### All combinations

Operators can do any of the following for unsupported operands

  - Cast
      - silent
      - with notice
      - with warning
      - causing a catchable error (fatal)
  - Notice + cast
  - Warning + cast
  - Throw Error
  - No operation

Please take a look at this [list of all combinations of operators and
operands](https://gist.github.com/jasny/bfd711844a8876f8206ed21357e2e2da).

## Proposal

By default, all PHP files are in weak type-checking mode for operators.
A new `declare()` directive is added, `strict_operators`, which takes
either `1` or `0`. If `1`, strict type-checking mode is used for
operators in the the file. If `0`, weak type-checking mode is used.

In strict type-checking mode, operators may cast operands to the
expected type. However:

  - Typecasting is not based on the type of the other operand
  - Typecasting is not based on the value of any of the operands
  - Operators will throw a `TypeError` for unsupported types

In case an operator can work with several (or all) types, the operands
need to match as no casting will be done by those operators.

The one exception is that [widening primitive
conversion](http://docs.oracle.com/javase/specs/jls/se7/html/jls-5.html#jls-5.1.2)
is allowed for `int` to `float`. This means that parameters that declare
`float` can also accept `int`.

``` php
declare(strict_operators=1);

1.2 + 2; // float(3.2)
```

In this case, we're passing an `int` to a function that accepts `float`.
The parameter is converted (widened) to float.

### Comparison operators

All comparison operators work with `int`, `float` and `bool` type
operands. The types of both operands need to match.

Strings, resources, arrays, objects and null only support the `==`,
`===`, `!=` and `!==` operators.

``` php
10 > 42;        // false
3.14 < 42;      // true

"foo" > "bar";  // TypeError("Unsupported type string for comparison operation")
"foo" > 10;     // TypeError("Operator type mismatch string and int for comparison operation")

"foo" == "bar"; // false
"foo" == 10;    // TypeError("Operator type mismatch string and int for comparison operation")
"foo" == null;  // TypeError("Operator type mismatch string and null for comparison operation")

true > false;   // true
true != 0;      // TypeError("Operator type mismatch bool and int for comparison operation")

[10] > [];      // TypeError("Unsupported type array for comparison operation")
[10] == [];     // false
```

The function of the `===` and `!==` operators remains unchanged.

#### Numeric string comparison

Numeric strings are compared the same way as non-numeric strings. To
compare two numeric strings as numbers, they need to be cast to integers
or floats.

``` php
"120" > "99.9";               // TypeError("Unsupported type string for comparison operation")
(float)"120" > (float)"99.9"; // true

"100" == "1e1";               // false
(int)"100" == (int)"1e2";     // true

"120" <=> "99.9";             // TypeError("Unsupported type string for comparison operation")
```

#### Array comparison

All comparison operators on arrays in strict modes will throw a
`TypeError`. This includes `==`.

Using `==` implies both comparing an array as an unsorted hashmap and
using type juggling. Allowing `==` for arrays ad disabling type juggling
makes `strict_operators` change the behavior of the operator.

``` php
$a = 0;
$b = "foo";
$a == $b;     // TypeError("Unsupported type string for comparison operation")
[$a] == [$b]; // TypeError("Unsupported type array for comparison operation")
```

The example above returns `true` without strict operators. Resulting in
`false` could be considered 'correct'. However if `$b` contains the
string `"0"` resulting in `false` would be an unexpected change in
behavior.

``` php
$a = 0;
$b = "0";
$a == $b;     // TypeError("Unsupported type string for comparison operation")
[$a] == [$b]; // TypeError("Unsupported type array for comparison operation")
```

The alternative of throwing a `TypeError` based on the contents of the
array goes against the principles of this RFC.

#### Object comparison

Comparing two objects of different classes using the `==` or `!=`
operator will throw a `TypeError`.

``` php
class Foo {
  public $x;
  
  public function __construct($x) {
    $this->x = $x;
  }
}

class FooBar extends Foo {}

(new Foo(10)) == (new Foo(10));     // true
(new Foo(10)) == (new Foo(99));     // false
(new Foo(10)) === (new Foo(10));    // false

(new Foo(10)) == (new FooBar(11));  // TypeError("Type mismatch Foo object and FooBar object for comparison operation")
(new Foo(10)) === (new FooBar(11)); // false
```

Comparing two objects of the same class will with these operators check
the properties of the objects. By default, properties are compared in a
similar fashion to the `===` operator. If the property of both objects
contains an object of the same class, they're compared as using the `==`
operator.

In the following examples, the `==` results in `false` when
`strict_operators` is used and `true` otherwise;

``` php
(new Foo(0)) == (new Foo('foo'));
(new Foo(10)) == (new Foo('10'));
(new Foo(null)) === (new Foo(0));
```

### Arithmetic operators

Arithmetic operators will only work with integers and floats. Using
operands of any other type will result in a `TypeError`.

In strict type-checking mode, the behavior of the operator is not
determined by the value of the operands. Thus for any string, including
numeric strings, a `TypeError` is thrown, so strings need to be
explicitly cast.

The `+` operator is still available for arrays as union operator,
requiring both values to be arrays.

### Incrementing/Decrementing operators

The incrementing/decrementing operators will throw a `TypeError` when
the operand is a string, boolean, null, array, object, or resource.

The function of these operators for integers and floats remains
unchanged.

### Bitwise Operators

Bitwise operators expect both parameters to be an integer. The `&`, `|`,
`^` and `~` operators also accept strings as operands.

Using strings for `>>` or `<<`, mixing strings with integers or using
any other type will throw a `TypeError`.

### String Operators

The concatenation operator `.` will throw a `TypeError` if any of the
operands is a boolean, array or resource. It will also throw a
`TypeError` if the operand is an object that doesn't implement the
`__toString()` method.

Integers, floats, null, and objects (with the `__toString()` method) are
cast to a string.

#### Variable parsing

When a string is specified in double quotes or with heredoc, variables
are parsed within it. Using the concatenation operator or double-quoted
string is considered interchangeable

### Logical Operators

The function of logical operators remains unchanged. All operands are
cast to booleans.

### Switch control structure

When strict-type checking for operators is enabled, the `switch`
statement will do a comparison similar to a comparison on arrays; Scalar
values in the array are compared using both type and value, thus similar
to the `===` operator. For arrays, the key order does not matter.
Objects of the same class will be compared similarly to the \`==\`
operator, while objects of different classes are always seen as not
equal. It will **never** throw a `TypeError`.

``` php
function match($value)
{
  switch ($value) {
    case ["foo" => 42, "bar" => 1]:
      return "foobar";
      break;
    case null:
      return "null";
      break;
    case 0:
      return "zero";
      break;
    case "1e1":
      return "1e1";
      break;
    case "10":
      return "10";
      break;
    default:
      throw new Exception("Unexpected value");
  }
}

match(["bar" => 1, "foo" => 42]); // "foobar"
match(0);                         // "zero"
match("10");                      // "10"
match("foo");                     // Exception("Unexpected value")
```

## Backward Incompatible Changes

Since the strict type-checking for operators is off by default and must
be explicitly used, it does not break backward-compatibility.

## Proposed PHP Version

This is proposed for PHP 8.0.

## FAQ

### What has been changed since the initial proposal?

  - Comparison operators `>`, `>=`, `<`, `<=` and `<=>` with string
    operands will throw a `TypeError`. [See
    motivation.](#why_aren_t_all_comparison_functions_available_for_strings)
  - Comparison operator `==` on array operands will always throw a
    `TypeError`.
  - Variable parsing in strings specified in double quotes and with
    heredoc, is also affected.
  - Concatenation operation (using `.`) on `null` does not throw a
    `TypeError`, but will cast `null` to an empty string.
  - There is no secondary vote for `switch`. If this RFC is accepted
    `strict_operators` will apply to `switch`.

### Why does

and \!= throw a TypeError instead of returning false? ==== In other
dynamically typed languages, like Python and Ruby, the `==` and `!=` do
a type check and always return `false` in case the type is different.
Throwing a `TypeError` is more common for statically typed languages
like Go.

The RFC tries to limit the cases where the behavior changes, rather than
that a `TypeError` is thrown, as much as possible. For instance `1 ==
true` and `null == 0` both evaluate to `true` without strict\_operators.
Having those statements be `false` based on the directive could lead to
unseen bugs.

### Why does the concatenation operator cast, but arithmetic operators don't?

The concatenation operator will cast integers, floats and *(when
`__toString` is defined)* objects to a string. This is a common use case
and operands are always typecasted to a string. Integers and floats
always have a proper string representation.

Arithmetic operators won't cast strings to an integer or float, because
not all strings can be properly represented as a number and a
`TypeError` must be thrown based on the operand type only, not the
value.

Both the concatenation operator and arithmetic operators throw a
`TypeError` for arrays, resources, and objects. Casting these to a
string or int/float doesn't give a proper representation of the value of
the operand.

Using a boolean or null as operand for both concatenation and arithmetic
operators also throws a `TypeError`. In most cases, the use of a boolean
or null indicates an error as many functions return `false` or `null` in
case of an error or when no result can be returned. This is different
from the function returning an empty string or `0`.
[`strpos`](https://php.net/strpos) is a well-known example.

### Will comparing a number to a numeric string work with strict operators?

No, this will throw a `TypeError`. Users that use string operators need
to explicitly typecast the string to an integer. If it concerns input
data, for instance from `$_POST`, it's recommended to use the [filter
functions](https://www.php.net/filter).

### Why aren't all comparison functions available for strings?

In many other languages, using `<`, `>`, `<=` and `=>` with string
operands performs an `strcmp` like operation. The `<=>` is even
described as strcmp as operators. Why do these operators throw an
exception with string operands when strict\_operators is enabled?

It's common to use these operators when both operands are numeric
strings. Having these statements return a different value based on the
directive, could lead to issues, especially when source code is
copy/pasted from an external codebase.

In case it concerns comparing text, it's better to use
`Collator::compare()`. For non-collation related strings, like date
strings, the `strcmp` function should be used.

Nikita Popov made the following argument:

\<blockquote\>Having `$str1 < $str2` perform a `strcmp()` style
comparison under strict\_operators is surprising. I think that overall
the use of lexicographical string comparisons is quite rare and should
be performed using an explicit `strcmp()` call. More likely than not,
writing `$str1 < $str2` is a bug and should generate a `TypeError`. Of
course, equality comparisons like `$str1 == $str2` should still work,
similar to the distinction you make for arrays.\</blockquote\>

### How can arrays be compared as unsorted hashmaps?

Arrays are sorted hashmaps in PHP but used as any type of collection
like unsorted maps, lists, and sets. Only comparing as unsorted hashmap
is supported using `==`, while other cases require the use of functions
like `sort()` and `array_values()`.

Strict comparison of arrays as unsorted hashmaps currently isn't
possible and requires sorting the array, prior to comparison.

With `==` unavailable when using `strict_operators`, sorting the array
would be the only option available.

Array functions might be added to compare arrays in different ways. But
that's outside the scope of this RFC.

### Why isn't is allowed to increment strings with strict\_operators?

Nikita Popov made the following argument:

\<blockquote\>String increment seems like a pretty niche use case, and I
believe that many people find the overflow behavior quite surprising. I
think it may be better to forbid string increment under
strict\_operators. \</blockquote\>

### Are built-in functions affected by strict\_operators?

No. Only operators (including the `case` statement)) in the source file
that has `declare(strict_operators=1)` are affected. Functions defined
elsewhere, including functions defined in extensions, are not affected.

Specifically `sort()` and `in_array()` will perform weak comparison as
usual and require the use of the `$sort_flags` and `$strict` arguments
to change the way the values are compared.

### Can relational operators be allowed for arrays?

If both arrays must have the same keys in the same order, using `<` or
`>` on two arrays can be useful. But, as shown in the examples, when
this is not the case, these type of comparisons will yield unexpected
results.

Throwing a `TypeError` only if the keys of the arrays don't match is not
in line with this RFC. The behavior of an operator should not depend on
the value of the operand, only on the type. Furthermore, a `TypeError`
would be misplaced here, as some arrays would be accepted but others
not, whereas a `TypeError` indicates no values of that type are
accepted.

### Are there cases where a statement doesn't throw a TypeError but yields a different result?

Yes, there are 3 cases where a comparison works differently in strict
operators mode.

``` 
  * Two numeric strings will be compared on equality as strings instead of numbers. Comparing two operands of the same type should always work. Under strict operators, the operation is only determined by the type and never by the value. Comparisons like ''"100" == "1e2"'' are unlikely to be intended and will return ''false'' with strict operators.
  * When comparing two objects no type juggling will be performed. This might result in ''false'' where ''true'' will be the result without strict operators.
  * With strict operators, the ''case'' in a ''switch'' statement will not do type conversion. It also doesn't compare numeric strings as numbers but as strings.
```

### Will this directive disable type juggling altogether?

No. Operators can still typecast under the given conditions. For
instance, the concatenation (`.`) operator will cast an integer or float
to a string, and boolean operators will cast any type of operand to a
boolean.

Typecasting is also done in other places: `if` and `while` statements
interpret expressions as boolean, and booleans and floats are cast to an
integer when used as array keys.

This RFC limits the scope to operators.

### Why is switch affected? It's not an operator.

Internally the `case` of a switch is handled as a comparison operator.
The issues with `case` are therefore similar (or even the same) to those
of comparison operators. The audience that `strict_operators` caters to,
likely want to get rid of this behavior completely.

## Unaffected PHP Functionality

This RFC

  - Does not affect any functionality concerning explicit typecasting.
  - Does not affect variable casting that occurs in (double-quoted)
    strings.
  - Is largely unaffected by other proposals like [PHP RFC: Saner string
    to number comparisons](/rfc/string_to_number_comparison) that focus
    on improving type juggling at the cost of breaking BC.

## Implementation

<https://github.com/php/php-src/pull/4375>

## Proposed Voting Choices

As this is a language change, a 2/3 majority is required.

The vote is a straight Yes/No vote for accepting the RFC and merging the
patch.

*Note: Voting on v1.0 of this RFC (targeting PHP 7.4) ended
prematurely.*