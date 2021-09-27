```python
class _const:
     class ConstError(TypeError):pass
     def __setattr__(self, name, value):
         if name in self.__dict__:
            raise self.ConstError(f"Can't rebind const {name} with {value}")
         else:
             self.__dict__[name] = value

import sys
sys.modules[__name__] = _const()
```

```python
def random_without_side_effect(seed=12343):
    pass
```
