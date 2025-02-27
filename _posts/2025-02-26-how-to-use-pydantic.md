---
toc: true
title: "pydantic 使用"
categories:
  - python
tags:
  - python
  - langchain
---

# Why use？

你还看的懂python代码吗?

https://github.com/django/django/blob/main/django/core/serializers/python.py

https://github.com/langchain-ai/langchain/blob/master/libs/langchain/langchain/agents/react/agent.py

- 支持类型提示
- **速度**- Pydantic的核心验证逻辑是用Rust编写的
- **JSON Schema**

如何将对象转为json? 如何将json转为object ?

如何将dict 转为 object

``` python
complex_dict = {
    'user': {
        'id': 12345,
        'name': 'John Doe',
        'email': 'john.doe@example.com',
        'address': {
            'street': '123 Main St',
            'city': 'Springfield',
            'zipcode': '12345'
        },
        'roles': ['admin', 'user'],
        'settings': {
            'theme': 'dark',
            'notifications': True
        },
        'history': [
            {'action': 'login', 'timestamp': '2025-02-01T10:00:00'},
            {'action': 'update_profile', 'timestamp': '2025-02-10T15:30:00'}
        ]
    }
}

# 访问数据
print(complex_dict['user']['name'])
print(complex_dict.get('user').get('name'))
print(complex_dict.get('user', {}).get('name', {}))

from pydantic import BaseModel, EmailStr
from typing import List, Dict
from datetime import datetime


class Address(BaseModel):
    street: str
    city: str
    zipcode: str


class HistoryItem(BaseModel):
    action: str
    timestamp: datetime


class Settings(BaseModel):
    theme: str
    notifications: bool


class User(BaseModel):
    id: int
    name: str
    email: EmailStr
    address: Address
    roles: List[str]
    settings: Settings
    history: List[HistoryItem]


class ComplexData(BaseModel):
    user: User


# 模拟验证
try:
    data = ComplexData(**complex_dict)
    print("数据验证通过：", data)
except Exception as e:
    print("数据验证失败：", e)

```

# Model

```python
# 定义一个对象, id 必须是int, name必须是str
class User(object):
    def __init__(self, id, name):
        if not isinstance(id, int):
            raise TypeError("id must be int")
        self.id = id
        if not isinstance(name, str):
            raise TypeError("name must be str")
        self.name = name

# 定义参数类型, 仅编辑器会提示错误
class User(object):
    def __init__(self, id: int, name: str):
        self.id = id
        self.name = name

```

## Basic model 

```python
from pydantic import BaseModel


class User(BaseModel):
    id: int
    name: str = 'Jane Doe'
```
在此示例中， User是一个具有两个字段的模型：


- id ，这是一个整数，是必需的
- name ，这是一个字符串，不需要（它具有默认值）。

模型实例可以使用model_dump方法序列化：
```json
{'id': 123, 'name': 'John Doe'}
```

### 模型方法

- model_validate()：验证给定对象是否符合 Pydantic 模型的要求
- model_validate_json()：验证给定的 JSON 数据是否符合 Pydantic 模型的要求。
- model_dump()：返回模型字段及其值的字典表示。
- model_dump_json()：返回 model_dump() 的 JSON 字符串表示。
- model_copy()：返回模型的副本（默认是浅拷贝）。
- model_json_schema()：返回表示模型 JSON Schema 的可序列化字典。


## Nested models

```python
from typing import List, Optional

from pydantic import BaseModel


class Foo(BaseModel):
    count: int
    size: Optional[float] = None


class Bar(BaseModel):
    apple: str = 'x'
    banana: str = 'y'


class Spam(BaseModel):
    foo: Foo
    bars: List[Bar]


m = Spam(foo={'count': 4}, bars=[{'apple': 'x1'}, {'apple': 'x2'}])
print(m)
"""
foo=Foo(count=4, size=None) bars=[Bar(apple='x1', banana='y'), Bar(apple='x2', banana='y')]
"""
print(m.model_dump())
"""
{
    'foo': {'count': 4, 'size': None},
    'bars': [{'apple': 'x1', 'banana': 'y'}, {'apple': 'x2', 'banana': 'y'}],
}
"""
```

## Validating data
- model_validate()：这个方法非常类似于模型的 __init__ 方法，不同的是它接受字典或对象，而不是关键字参数。如果传入的对象无法验证，或者它既不是字典也不是目标模型的实例，将会引发 ValidationError。

- model_validate_json()：此方法验证提供的 JSON 字符串或字节对象。如果你的输入数据是 JSON 数据，通常建议使用此方法，因为它比手动解析数据为字典的方式更高效。你可以在文档的 JSON 部分了解更多关于 JSON 解析的内容。

- model_validate_strings()：这个方法接受一个字典（可以是嵌套的），字典的键和值都是字符串，并在 JSON 模式下验证数据，以便将这些字符串强制转换为正确的类型。

```python
from datetime import datetime
from typing import Optional

from pydantic import BaseModel, ValidationError


class User(BaseModel):
    id: int
    name: str = 'John Doe'
    signup_ts: Optional[datetime] = None


m = User.model_validate({'id': 123, 'name': 'James'})
print(m)

try:
    User.model_validate(['not', 'a', 'dict'])
except ValidationError as e:
    print(e)
    """
    1 validation error for User
      Input should be a valid dictionary or instance of User [type=model_type, input_value=['not', 'a', 'dict'], input_type=list]
    """
#
m = User.model_validate_json('{"id": 123, "name": "James"}')
print(m)
m = User.model_validate_strings({'id': '123', 'name': 'James'})
print(m)
m = User.model_validate_strings(
    {'id': '123', 'name': 'James', 'signup_ts': '2024-04-01T12:00:00'}
)
print(m)
```

## Generic models 

创建通用的http响应
```json
{"code": 200, "message": "ok", "data": {"name": "foo"}}
{"code": 200, "message": "ok", "data": [1,2,3]}
{"code": 200, "message": "ok", "data": [{}, {}]
```

```python
from typing import Generic, List, Optional, TypeVar

from pydantic import BaseModel

DataT = TypeVar('DataT')


class DataModel(BaseModel):
    numbers: List[int]
    people: List[str]


class Response(BaseModel, Generic[DataT]):
    code: int = 200
    message: str = 'OK'
    data: Optional[DataT] = None


print(Response[int](data=1))
print(Response[int](data=1).model_dump_json())

print(Response[List[int]](data=[1, 2, 3]))
print(Response[List[int]](data=[1, 2, 3]).model_dump_json())

print(Response[DataModel](data=DataModel(numbers=[1, 2, 3], people=['Alice', 'Bob'])))
print(Response[DataModel](data=DataModel(numbers=[1, 2, 3], people=['Alice', 'Bob'])).model_dump_json())

```
## RootModel

封装单一的数据结构

```python
from pydantic import RootModel
from typing import List

Ids = RootModel[List[int]]

ids = Ids([1, 2, 3])
print(ids.model_dump_json())

for i in ids:
    print(i)


class IdsModel(RootModel):
    root: List[int]

    def __iter__(self):
        return iter(self.root)

    def __getitem__(self, item):
        return self.root[item]


ids = IdsModel([1, 2, 3])
print(ids)
for i in ids:
    print(i)
```
## 不可变性

实例的属性值不可以被更改，例如配置文件
```python
from pydantic import BaseModel, ConfigDict, ValidationError


class FooBarModel(BaseModel):
    model_config = ConfigDict(frozen=True)

    a: str
    b: dict


foobar = FooBarModel(a='hello', b={'apple': 'pear'})

try:
    foobar.a = 'different'
except ValidationError as e:
    print(e)
```


## 额外的字段

通常情况下赋值未定义的字段是被允许的
```python
class User(object):
    def __init__(self, name, age):
        self.name = name
        self.age = age


user = User('Max', 27)
print(user.name)
user.sex = 'male'
print(user.sex)
```

```python
from pydantic import BaseModel, ConfigDict, ValidationError


class User(BaseModel):
    name: str
    age: int

    model_config = ConfigDict(extra='forbid')

# allow 额外的字段
user = User(name='John', age=30)
print(user)

user.sex = 'male'


class User(BaseModel):
    name: str
    age: int

    model_config = ConfigDict(extra='allow')


user = User(name='John', age=30)
print(user)

user.sex = 'male'
print(user.__pydantic_extra__)
```

# Field

https://www.django-rest-framework.org/tutorial/1-serialization/#creating-a-serializer-class

## Default values

为字段设置默认值
``` python
from uuid import uuid4

from pydantic import BaseModel, Field, EmailStr


class User(BaseModel):
    id: str = Field(default_factory=lambda: uuid4().hex)
    name: str = Field(default='John Doe')
    email: EmailStr
    username: str = Field(default_factory=lambda data: data['email'])


user = User(email='john@doe.com')
print(user)
# > name='John Doe'

```
## Field aliases
```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(alias='username')


user = User(username='johndoe')
print(user)
# > name='johndoe'
print(user.model_dump(by_alias=True))
# > {'username': 'johndoe'}

```

## 约束
### Numeric Constraints
```python
from pydantic import BaseModel, Field


class Product(BaseModel):
    name: str
    price: float = Field(ge=0, multiple_of=0.1)  # 价格必须大于等于 0，并且是 0.1 的倍数
    stock: int = Field(ge=0)  # 库存必须大于等于 0
    max_order: int = Field(le=10)  # 最大订单数不能超过 10


# 创建实例
product = Product(name="Product A", price=9.99, stock=100, max_order=5)
print(product)

```
### String Constraints
```python
from pydantic import BaseModel, Field


class Foo(BaseModel):
    short: str = Field(min_length=3)
    long: str = Field(max_length=10)
    regex: str = Field(pattern=r'^\d*$')  


foo = Foo(short='foo', long='foobarbaz', regex='123')
print(foo)
#> short='foo' long='foobarbaz' regex='123'
```

## Strict Mode

不对类型做自动转换

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(strict=True)  
    age: int = Field(strict=False)


user = User(name='John', age='42')  
print(user)
#> name='John' age=42
```

## Exclude
```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str
    age: int = Field(exclude=True)


user = User(name='John', age=42)
print(user.model_dump())  
#> {'name': 'John'}
```

## Deprecated fields

表名此字段已废弃, 但是可以正常访问

```python
from typing_extensions import Annotated

from pydantic import BaseModel, Field


class Model(BaseModel):
    deprecated_field: Annotated[int, Field(deprecated=True)]


m = Model(deprecated_field=1)
print(m.deprecated_field)

```

## Computed Field
```python
from pydantic import BaseModel, computed_field


class Box(BaseModel):
    width: float
    height: float
    depth: float

    @computed_field
    @property  
    def volume(self) -> float:
        return self.width * self.height * self.depth


b = Box(width=1, height=2, depth=3)
print(b.model_dump())
#> {'width': 1.0, 'height': 2.0, 'depth': 3.0, 'volume': 6.0}
```
# JSON Schema

生成符合 openapi 规范的json描述文件

```python
import json
from typing import Union

from pydantic import BaseModel, TypeAdapter


class Cat(BaseModel):
    name: str
    color: str


class Dog(BaseModel):
    name: str
    breed: str


ta = TypeAdapter(Union[Cat, Dog])
ta_schema = ta.json_schema()
print(json.dumps(ta_schema, indent=2))
```

自定义
```python
import json

from pydantic import BaseModel, EmailStr, Field, SecretStr


class User(BaseModel):
    age: int = Field(description='Age of the user')
    email: EmailStr = Field(examples=['marcelo@mail.com'])
    name: str = Field(title='Username')
    password: SecretStr = Field(
        json_schema_extra={
            'title': 'Password',
            'description': 'Password of the user',
            'examples': ['123456'],
        }
    )


print(json.dumps(User.model_json_schema(), indent=2))
```

# Union
- 左至右模式- 最简单的方法，尝试按顺序尝试联合的每个成员，并返回第一匹配
- 智能模式- 类似于“从左到右模式”的成员类似；但是，验证将经过第一场比赛以尝试找到更好的匹配，这是大多数联合验证的默认模式
- 歧视工会- 基于歧视者，只有一名工会成员受到尝试

## Left to Right 
```python
from typing import Union

from pydantic import BaseModel, Field, ValidationError


class User(BaseModel):
    id: Union[str, int] = Field(union_mode='left_to_right')


print(User(id=123))
#> id=123
print(User(id='hello'))
#> id='hello'

try:
    User(id=[])
except ValidationError as e:
    print(e)
    """
    2 validation errors for User
    id.str
      Input should be a valid string [type=string_type, input_value=[], input_type=list]
    id.int
      Input should be a valid integer [type=int_type, input_value=[], input_type=list]
    """
```
## Smart

智能模式

```python
from typing import Union

from pydantic import BaseModel, Field


class User(BaseModel):
    id: Union[int, str]


class UserLeftToRight(BaseModel):
    id: Union[int, str] = Field(union_mode='left_to_right')

user = User(id=123)
print(user, type(user.id))
user = UserLeftToRight(id=123)
print(user, type(user.id))

user_01 = User(id='123')
print(user_01, type(user_01.id))
user_02 = UserLeftToRight(id='123')
print(user_02, type(user_02.id))
```

## Discriminated

受歧视的

``` python
from typing import Literal, Union

from pydantic import BaseModel, Field, ValidationError


class Login(BaseModel):
    req_type: Literal['login']
    username: str
    password: str


class Register(BaseModel):
    req_type: Literal['register']
    email: str
    password: str


class Logout(BaseModel):
    req_type: Literal['logout']
    token: str


class UserRequest(BaseModel):
    action: str
    data: Union[Login, Register, Logout] = Field(discriminator="req_type")


login_request_data = {
    "action": "login",
    "data": {
        "req_type": "login",
        "username": "johndoe",
        "password": "secret"
    }
}

register_request_data = {
    "action": "register",
    "data": {
        "req_type": "register",
        "email": "johndoe@example.com",
        "password": "secret"
    }
}

logout_request_data = {
    "action": "logout",
    "data": {
        "req_type": "logout",
        "token": "<token>"
    }
}

try:
    login_request = UserRequest(**login_request_data)
    print("登录请求完成", login_request)
    register_request = UserRequest(**register_request_data)
    print("注册请求完成", register_request)
    logout_request = UserRequest(**logout_request_data)
    print("登出请求完成", logout_request)
except ValidationError as e:
    print(e)

```

# Validators
Internal Validation（内部验证）
Pydantic 会在模型的字段赋值时自动执行一系列验证。这些内部验证包括：

- 数据类型检查：确保字段的值符合模型定义的类型。
- 字段约束：例如，最小值、最大值、正则表达式等约束检查。
- 必须字段：确保必填字段有值。

## Field validators
### After validators
表示该验证器会在字段的原始输入数据被解析成目标类型后触发。这种模式适用于需要基于已处理后的值（而非原始输入）进行验证的场景，尤其是当验证逻辑依赖字段的类型转换结果时。


```python
from pydantic import BaseModel, ValidationError, field_validator


class Model(BaseModel):
    number: int

    @field_validator('number', mode='after')  
    @classmethod
    def is_even(cls, value: int) -> int:
        if value % 2 == 1:
            raise ValueError(f'{value} is not an even number')
        return value  


try:
    Model(number=1)
except ValidationError as err:
    print(err)
    """
    1 validation error for Model
    number
      Value error, 1 is not an even number [type=value_error, input_value=1, input_type=int]
    """
```
### Before validators
在 Pydantic 的内部解析和验证之前运行
mode='before' 可以让你在字段赋值之前对数据进行预处理或验证，确保数据符合预期的格式或结构。这对于需要清理或标准化数据的场景非常有用。
```python
from typing import Any, List

from pydantic import BaseModel, ValidationError, field_validator


class Model(BaseModel):
    numbers: List[int]

    @field_validator('numbers', mode='before')
    @classmethod
    def ensure_list(cls, value: Any) -> Any:  
        if not isinstance(value, list):  
            return [value]
        else:
            return value


print(Model(numbers=2))
#> numbers=[2]
try:
    Model(numbers='str')
except ValidationError as err:
    print(err)  
    """
    1 validation error for Model
    numbers.0
      Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='str', input_type=str]
    """
```

### Plain validators

mode='plain' 主要用于以下场景：

- 跳过 Pydantic 的类型验证：如果你确定字段值的类型已经符合预期，不需要再执行 Pydantic 内部的类型检查，可以使用 mode='plain' 来减少开销。

- 早期终止验证：当你希望在验证过程中尽早退出，不让 Pydantic 继续进行其他验证时，使用 mode='plain' 可以立刻返回并停止后续验证
```python
from typing import Any

from pydantic import BaseModel, field_validator


class Model(BaseModel):
    number: int

    @field_validator('number', mode='plain')
    @classmethod
    def val_number(cls, value: Any) -> Any:
        if isinstance(value, int):
            return value * 2
        else:
            return value


print(Model(number=4))
#> number=8
print(Model(number='invalid'))  
#> number='invalid'
```

### Wrap validators
在 Pydantic 中，Wrap Validators 是最灵活的一种验证器类型。你可以选择在 Pydantic 和其他验证器处理输入之前或之后执行代码，甚至可以通过立即返回值或抛出错误来终止验证过程。

Wrap Validators 需要通过一个额外的参数——handler 来进行定义。这个参数是一个可调用的函数，它接收需要验证的值作为参数。通过这种方式，你可以在验证过程中对 Pydantic 的行为进行包装、控制或扩展。

工作原理
- before 处理：你可以在 Pydantic 执行默认的类型验证之前执行自定义逻辑。
- after 处理：你可以在 Pydantic 完成验证后对结果进行处理。
- 终止验证：你可以通过返回一个提前的值来终止验证，或者通过抛出异常来停止进一步的验证过程。

这种方式提供了极大的灵活性，允许你控制验证的顺序、时机和逻辑。

```python
from typing import Any

from typing_extensions import Annotated

from pydantic import BaseModel, Field, ValidationError, ValidatorFunctionWrapHandler, field_validator


class Model(BaseModel):
    my_string: Annotated[str, Field(max_length=5)]

    @field_validator('my_string', mode='wrap')
    @classmethod
    def truncate(cls, value: Any, handler: ValidatorFunctionWrapHandler) -> str:
        try:
            return handler(value)
        except ValidationError as err:
            if err.errors()[0]['type'] == 'string_too_long':
                return handler(value[:5])
            else:
                raise


print(Model(my_string='abcde'))
#> my_string='abcde'
print(Model(my_string='abcdef'))
#> my_string='abcde'
```

| 验证模式   | 触发时机                 | 特点                           | 使用场景 |
|------------|--------------------------|--------------------------------|----------|
| **`before`** | 在 Pydantic 类型验证之前 | 用于数据清洗和转换              | 数据转换、格式化、预处理 |
| **`after`**  | 在 Pydantic 类型验证之后 | 用于后处理和额外的业务逻辑验证 | 依赖字段验证、复杂逻辑验证 |
| **`plain`**  | 直接终止验证流程         | 简单的逻辑验证，跳过类型验证    | 快速验证、跳过类型验证、业务规则检查 |
| **`wrap`**   | 可以在验证前后执行       | 灵活控制验证流程，包装自定义逻辑 | 包装验证、复杂的验证控制、日志记录等 |

## Model validators 

### After validators
一种用于在整个模型验证完成后执行额外逻辑的验证器，它通常作为实例方法定义，并且可以被看作是模型的后初始化钩子（post-initialization hooks）。这一模式使得你可以在字段验证后执行更复杂的验证逻辑或额外操作。
```python
from typing_extensions import Self

from pydantic import BaseModel, model_validator


class UserModel(BaseModel):
    username: str
    password: str
    password_repeat: str

    @model_validator(mode='after')
    def check_passwords_match(self) -> Self:
        if self.password != self.password_repeat:
            raise ValueError('Passwords do not match')
        return self
```

### Before validators

运行在模型实例化之前，这使得它比 after 验证器更灵活。由于它在模型实例化之前执行，它处理的是原始输入数据，这些数据理论上可能是任何类型的对象（例如字典、列表、字符串等）。这意味着，before 验证器必须处理和验证更原始的输入，而不是已经通过 Pydantic 的类型验证的数据。

``` python
from typing import Any

from pydantic import BaseModel, model_validator


class UserModel(BaseModel):
    username: str

    @model_validator(mode='before')
    @classmethod
    def check_card_number_not_present(cls, data: Any) -> Any:
        if isinstance(data, dict):
            if 'card_number' in data:
                raise ValueError("'card_number' should not be included")
        return data

    # @model_validator(mode='before')
    # @classmethod
    # def strip_username(cls, data: Any) -> Any:
    #     if isinstance(data, dict):
    #         if 'username' in data:
    #             data['username'] = data['username'].strip()
    #     return data

user = UserModel(username='test21111         ')
print(user)
```
### Wrap validators
是 Pydantic 中最灵活的验证方式，它允许你在 Pydantic 和其他验证器处理输入数据之前或之后运行代码。你还可以立即终止验证操作，方法是提前返回数据或通过抛出错误来阻止进一步的验证。

| **验证器**         | **运行时机**                         | **处理数据**             | **使用场景**                                                                                      |
|--------------------|--------------------------------------|--------------------------|--------------------------------------------------------------------------------------------------|
| **Before**         | 在模型实例化之前                    | 原始输入数据（未经过验证）| 数据清理、格式转换、基础验证；检查必填字段、转换字段类型等                                        |
| **After**          | 在字段验证后，模型实例化后          | 已经经过验证的数据       | 复杂验证、模型级别修复、业务规则验证；检查多个字段之间的关系，或做最终修复                      |
| **Wrap**           | 在任何阶段（验证前后）               | 可以修改字段，处理原始数据或验证结果 | 灵活的条件验证、强制数据转换、提前返回修改过的数据、根据条件决定验证流程                       |

## Validation context

https://gitlab.dxy.net/dev-ops/friday/django-cmdb/-/blob/master/cmdb/application/serializer.py#L46

您可以将上下文对象传递到[验证方法](https://docs.pydantic.dev/latest/concepts/models/#validating-data)，可以使用[`context`](https://docs.pydantic.dev/latest/api/pydantic_core_schema/#pydantic_core.core_schema.ValidationInfo.context)属性在验证器函数中访问该方法

``` python
from pydantic import BaseModel, ValidationInfo, field_validator


class Model(BaseModel):
    text: str

    @field_validator('text', mode='after')
    @classmethod
    def remove_stopwords(cls, v: str, info: ValidationInfo) -> str:
        if isinstance(info.context, dict):
            stopwords = info.context.get('stopwords', set())
            v = ' '.join(w for w in v.split() if w.lower() not in stopwords)
        return v


data = {'text': 'This is an example document'}
print(Model.model_validate(data))  # no context
#> text='This is an example document'
print(Model.model_validate(data, context={'stopwords': ['this', 'is', 'an']}))
#> text='example document'
```

# Settings Management

提供了可选的Pydantic功能，用于从环境变量或秘密文件中加载设置或配置类。

``` shell
pip install pydantic-settings
```

```py
from pydantic_settings import BaseSettings, SettingsConfigDict


class AppConfig(BaseSettings):
    model_config = SettingsConfigDict(env_file='.env', env_file_encoding='utf-8', env_prefix='my_prefix_')

    app_name: str
    app_env: str
    database_url: str
    debug: bool


app = AppConfig()
print(app)
```

环境变量始终由于.env

