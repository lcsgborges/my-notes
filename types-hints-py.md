# Types Hints Python (Dicas de tipos)

> Ambiente com `python>=3.13`

> Também chamado de Type annotations

Os types hints não alteram por padrão o comportamento do interpretador do Python, entretanto, servem para coisas muito útis, como:
- Permitir que ferramentas como Mypy ou Pyright encontrem erros antes de rodar
- Ativar autocomplete inteligente e mostrar docs nos editores
- Ajudar a entender o código
- Evitar bugs bobos (ex: passar int onde era str, ...)
- Melhorar a legibilidade do código, como se fosse uma documentação automática

## Tipo básicos (explícitos  implícitos)

```python
# Tipagem explícita
name: str = "Lucas Borges"
x: int = 10
y: float = 20.5
is_valid: bool = False
data: bytes = b"weather"
```

## Constantes

Constantes não constumam ser reatribuídas, então a tipagem é redundante. O próprio valor já deixa claro o tipo

```python
CONSTANT = "any value" # CONSTANTE: Literal["any value"]
```

## Coleções

Geralmente tipos com mais de um valor (listas, tuplas, dicionários, ...):

```python
list_nums: list[int] = [1, 2, 3]
tuple_two_values: tuple[str, int] = ("value", 123)
tuple_many_values: tuple[str, ...] = "a", "b", "c", "d", "..."
any_set: set[int] = {1, 2, 3, 4}
fronze_set: frozenset[int] = frozenset([5, 6, 7])
any_dict:  dict[str, str] = {"key1": "value1", "key2": "value2"}
any_range: range = range(5)
```

## Outros tipos

```python
nothing: None = None
anything: object = "123"
any_type: type = int
```

## Constantes com Final

```python
from typing import Final

CONSTANT_FINAL: Final[list[str]] = ["a", "b", "c"]
```
