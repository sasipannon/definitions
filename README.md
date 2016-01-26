definitions
===========

Load and validate YAML definitions against a schema. Classes are
directly instantiated from names in your definition files.

Definition
----------

The only reserved key in definitions is `type` which can hold the name of a
subclass of the the type defined in the schema. Now, let's take a look at the
schema.

```yaml
key1: value
key2: ClassName
key3:
  argument1: foo
  argument2: 42
key4:
  type: SubClassName
  argument1: foo
  argument2: 42
```

Schema
------

Schemas are defined in a nested way. For example, you can specify the schemas
for required keys of a dict or for constructor arguments of a class. The
following keys have a special meaning.

| Key | Description |
| --- | ----------- |
| `type` | Name of Python type Name of the Python type or base class. |
| `module` | Where to import non-primitive types from. |
| `default` | Parsed when the key is not specified. |
| `arguments` | Mapping or single nested schema describing constructor arguments. |
| `elements` | List or single nested schema describing valid elements that are passed as a collection to the constructor. |

Only one of `arguments` and `elements` can be specified at the same time.

Example
-------

### Definition

```yaml
type: dict
collection: True
arguments:
  cost: SquaredError
  constraints:
  - angle: 70
  - type: HINGE
    angle: 120
  distribution:
    type: Gaussian
    variance: 2.5
```

### Schema

```yaml
cost:
  type: Cost
  module: mypackage.cost
constraints:
  type: list
  elements:
    type: Constraint
    module: mypackage.constraint
    arguments:
      constraint_type: ConstraintType
      module: mypackage.constraint
distribution:
  type: Distribution
  module: mypackage.distribution
  arguments:
    mean: float
    variance: float
backup:
  type: bool
  default: false
```

### Usage in Code

```python
from definitions import Parser
from mypackage.cost import SquaredError
from mypackage.contraint import Constraint, ConstraintType
from mypackage.distribution import Gaussian


parser = Parser('schema.yaml', attribute_dicts=True)
definition = parser('definition.yaml')

assert isinstance(definition.cost, SquaredError)

first = definition.constraints[0]
assert isinstance(first, Constraint)
assert first.get_angle() == 70
assert first.get_type() == ConstraintType.ROTARY  # Constructor default

second = definition.constraints[1]
assert isinstance(second, Constraint)
assert second.get_angle() == 120
assert second.get_type() == ConstraintType.HINGE

assert isinstace(definition.distribution, Gaussian)
assert definition.distribution.get_mean() == 0
assert definition.distribution.get_variance() == 2.5

assert not definition.backup
```

Advanced Features
-----------------

### Collection of arbitrary types

Don't specify the `elements` key of a list or the `arguments` of a dict.

```yaml
type: list
```

### Enumerations

Usually, types get instantiated. However, for enumerations the corresponding
constant is stored instead. Just use a `type` that inherits from Python 3's
`enum`.

### Access items in dict as properties

Passing `attribute_dicts=True` to the parser consturctor substitutes all `dict`
types with `AttributeDict`. This is just a Python dict except keys can be
accessed as attributes. You can also explicitly use this type in the schema to
allow attribute access only for some of the dicts.

```python
definition = Parser('schema.yaml')('definition.yaml')
assert definition['key'] == value

definition = Parser('schema.yaml', attribute_dicts=True)('definition.yaml')
assert definition.key == value
```

### Defaults for constructor arguments

It is recommended to define arguments defaults in the constructor of the base
class. However, for standard types, it can make sense to define argument
defaults.

```yaml
type: ArgumentParser
module: argparse
arguments:
  prog: Program name.
  description: Description.
```

### Unknown arguments

Keys without a special meaning will always be passed as keyword arguments to
the constructur of the type. This allows subclasses to accept additional
parameters that are not listed in the base constructor.

### Types rather than instances

Usually, it turns out that working on instance objects is a better design than
working on type objects. However, it is possible. Note that the object will
first be instantiated and the type inferred afterwards.

```yaml
type: type
arguments:
    type: BaseClass
    module: mypackage
```

For arbitraty types, this becomes quite simple.

```yaml
type: type
```

### Callables as types

The `type` key works with any callable, not just constructors. The return value
of the callable will be stored under the key.

```yaml
filter:
  type: compile
  module: regex
```

```yaml
filter: '.*'
```
