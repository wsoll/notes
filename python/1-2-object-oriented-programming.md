- [Inheritance](#inheritance)
  * [Diamon Inheritance Problem](#diamon-inheritance-problem)
- [Metaclasses](#metaclasses)
  * [Auto class registration](#auto-class-registration)
  * [Enforcing class implementation / pattern](#enforcing-class-implementation---pattern)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


# Inheritance
...

## Diamon Inheritance Problem
Programming languages that allow only single inheritance don't have the problem. Anyway Python allows multiple inheritance...:

```python
class A:
    def __init__(self):
        print("A")


class B(A):
    def __init__(self):
        print("B")
        super().__init__()


class C(A):
    def __init__(self):
        print("C")
        super().__init__()


class D(B, C):
    def __init__(self):
        print("D")
        super().__init__()


if __name__ == "__main__":
    d = D()
```

```commandline
D
B
C
A
```

1. It is recommended to use super() builtin to do not encounter the issue or duplicate initialization (for initializing parents manually).
2. Order of inheritance for subclass defines initialization order. 


# Metaclasses

Just like class is objects factory, a metaclass is classes factory. In Python everything
is an object (there is not actual 'primitive' types) and its actual implementation 
contains many classes and metaclasses. Metaclasses are a fundamental part of Python's 
type system.

Use cases e.g.:

1. Auto class registration.
2. Enforcing classes implementations.

## Auto class registration

```Python
class PluginRegistry(type):
    def __new__(cls, name, bases, dct):
        new_class = super().__new__(cls, name, bases, dct)
        if not hasattr(cls, 'plugins'):
            cls.plugins = []
        else:
            cls.plugins.append(new_class)
        return new_class

    @classmethod
    def get_plugins(cls):
        return cls.plugins


class PluginBase(metaclass=PluginRegistry):
    pass


class PluginOne(PluginBase):
    def run(self):
        print("PluginOne running")


class PluginTwo(PluginBase):
    def run(self):
        print("PluginTwo running")


plugin1 = PluginOne()
plugin2 = PluginTwo()

registered_plugins = PluginRegistry.get_plugins()
print("Registered Plugins:")
for plugin in registered_plugins:
    print(plugin.__name__)

```

## Enforcing class implementation / pattern

```python
class EndpointRegistrationMeta(type):
    def __new__(cls, name, bases, dct):
        # Check if the class defines an 'endpoints' attribute
        if 'endpoints' not in dct:
            raise ValueError(f"Class '{name}' must define 'endpoints' attribute for endpoint registration")
        
        # Validate the 'endpoints' attribute
        endpoints = dct['endpoints']
        if not isinstance(endpoints, dict):
            raise ValueError("Endpoints must be defined as a dictionary")

        for endpoint_name, endpoint_func in endpoints.items():
            # Check if each endpoint is a callable function
            if not callable(endpoint_func):
                raise ValueError(f"Invalid endpoint '{endpoint_name}'. Endpoints must be callable functions")

            # Check if endpoint name follows snake_case convention
            if not endpoint_name.islower() or '_' in endpoint_name:
                raise ValueError(f"Invalid endpoint name '{endpoint_name}'. Endpoint names must be in snake_case")

        return super().__new__(cls, name, bases, dct)

# Example HTTP client class enforcing endpoint registration
class HTTPClient(metaclass=EndpointRegistrationMeta):
    endpoints = {}

    @classmethod
    def register_endpoint(cls, name, func):
        cls.endpoints[name] = func

# Usage example
class MyHTTPClient(HTTPClient):
    endpoints = {
        'get_user': lambda user_id: f"GET /users/{user_id}",
        'create_user': lambda data: f"POST /users - {data}",
        # Invalid endpoint name
        'InvalidEndpoint': lambda: "Invalid endpoint"
    }

```