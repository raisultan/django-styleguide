- [Общее](#общее)
  - [PEP8](#pep8)
  - [Импорты](#импорты)
    - [Порядок импортов](#порядок-импортов)
  - [Комментарии](#комментарии)
    - [Однострочные докстринги](#однострочные-докстринги)
    - [Многострочные докстринги](#многострочные-докстринги)
    - [Inline комментарии](#inline-комментарии)
  - [Строковые литералы и форматирование](#строковые-литералы-и-форматирование)
    - [Метод format:](#метод-format)
    - [f-strings:](#f-strings)
  - [Аннотации типов](#аннотации-типов)
    - [Соответствие типов в аннотациях](#соответствие-типов-в-аннотациях)
      - [Пример:](#пример)
  - [Разворачивание скобок и запятые](#разворачивание-скобок-и-запятые)
    - [Примеры:](#примеры)
  - [Именованные аргументы (kwargs)](#именованные-аргументы-kwargs)
  - [Инициализаторы пакетов - __init__.py](#инициализаторы-пакетов---initpy)
- [Модели](#модели)
  - [Валидация моделей](#валидация-моделей)
  - [Свойства моделей (@property)](#свойства-моделей-property)
- [Services & selectors](#services--selectors)
  - [Fat models, thin views](#fat-models-thin-views)
  - [Бизнес-логика в сервисах и селекторах](#бизнес-логика-в-сервисах-и-селекторах)
  - [Наш подход к написанию и хранению бизнес-логики](#наш-подход-к-написанию-и-хранению-бизнес-логики)
  - [Нейминг сервисов](#нейминг-сервисов)
  - [Нейминг методов](#нейминг-методов)

## Общее

**Перед прочтением стайлгайда, настоятельно рекомендуем ознакомиться с:**
* [PEP8](https://www.python.org/dev/peps/pep-0008/)
* [Рекомендации по стилю кода от Django](https://docs.djangoproject.com/en/dev/internals/contributing/writing-code/coding-style/)

**В Django, бизнес-логика должна быть в:**
* В сервисах, которые предназначены именно для этого.
* В методе `clean` модели, для дополнительной/кастомной валидации (с некоторыми исключениями).
* В свойствах (`property`) модели (с некоторыми исключениями).

**В Django, бизнес-логика не должна быть в:**
* В контроллерах - Views, ViewSets.
* В сериалайзерах и формах.
* В методе `save` модели.

### PEP8
Придерживаемся основной части соглашений PEP8. Исключением из правил, является длина строк, в нашем случае это 100 символов.

### Импорты
Порядок импортов, так же производится по PEP8. Единственное, на что необходимо обратить внимание, это то, что при импорте зависимостей внутри одной Django App стоит использовать [explicit relative import](https://learndjango.com/tutorials/django-best-practices-imports). Такой выбор обусловлен тем, что данный стиль импортов открыто "говорит", какие импортируемые зависимости являются внутренними, а какие внешними, что в свою очередь может понадобиться при портировании модулей и пакетов.

Также, вынужденным исключением из правил, при импорте зависимостей, может стать циклический импорт. В таких случаях, позволяется импортировать зависимость в нужной функции или в методе.

#### Порядок импортов
Стиль импортов должен соответствовать PEP8:

```python

# стандартная библиотека
from typing import Final, Tuple

# 3rd party зависимости
from constance import config
from django.conf import settings
from django.core.cache import cache
from django.utils.translation import ugettext_lazy as _
from rest_framework_simplejwt.tokens import RefreshToken

# внутренние зависимости
from apps.common.tasks import send_sms
from ..constants import SecondFactorAuthType
from ..models import User
from .throttle import ThrottleService
```

Также, мультилайн импорты оформляются как `multiline hanging indentation` с конечной запятой после последнего элемента:

```python
from django.contrib.auth.models import Group

from ..models import (
    BaseFee, FixedFee, PercentageFee, Transaction, Wallet,
)
from .wallet_balance import WalletBalanceService
```

Для автоматической сортировки импортов, предлагается использовать [isort](https://github.com/PyCQA/isort).

### Комментарии
Сам по себе, код должен быть понятен и самодокументируемым. Комментарии необходимы, где они реально необходимы и к месту. Где происходит что-то неочевидное, edge-кейсы бизнес-логики и т.д.

Если такой кейс произошел, и рефакторинг не поможет, то ниже продемонстрированы примеры:

#### Однострочные докстринги
```python
"""Сервис создания транзакций."""
```

#### Многострочные докстринги
```python
"""
Сервис создания транзакций.

При транзакции вывода, сумма транзакции не должна превышать текущий баланс юзера.
"""
```

#### Inline комментарии

Нужно оставлять при неявных случаях реализации бизнес-логики, при ссылке на тикет или тред в stackoverflow.com например.

```python
# невозможно выполнить аннотирование нескольких полей: ссылка на issue в forum.djangoproject
```

### Строковые литералы и форматирование
При кейсе, если строка предусматривает использование в качестве шаблона, для описания исключения например, то используется [метод .format()](https://docs.python.org/3.8/library/string.html#string.Formatter.format). В любом другом случае, используются [f-строки](https://docs.python.org/3/glossary.html#term-f-string).

#### Метод format:
```python
class TransactionError:
	INSUFFICIENT_FUNDS = 'Insufficient funds on sender wallet, {amount}{currency} needed'

# использование
TransactionError.INSUFFICIENT_FUNDS.format(amount=189.0, currency='EUR')
```

#### f-strings:
```python
def __str__(self) -> str:
	return f'Transaction {self.id}: {self.amount}{self.amount_currency}'
```

### Аннотации типов
На текущий момент, проект не имеет стейдж тайп-чекинга. Тем не менее, обуславливаясь тем, что современные IDE поддерживают тайп-чекинг, в том или ином виде, было решено аннотировать все функции, методы и константы (в некоторых случаях) в проекте. Исходя из этого, любая функция, метод и константа, в случае если она является примитивом, должны быть покрыты аннотациями типов.

#### Соответствие типов в аннотациях
В некоторых кейсах, позволяется пренебречь точностью типов в аннотациях. Одним из таких может стать избежание циклических импортов, например в сервисе кастомной валидации.

##### Пример:

Модель транзакции:
```python
class Transaction(models.Model):
	created_at = models.DateTimeField()
	amount = MoneyField()
	...

	def clean(self) -> None:
		super().clean()
		TransactionValidationService.validate(transaction=self)
```

Сервис валидации:
```python
class TransactionValidationService:
	@staticmethod
	def validate(transaction: Any) -> None: # несоответствие с фактическим типом
		...
```
В этом случае, при попытке импорта модели транзакции в сервис валидации, произойдет `ImportError`, а именно ошибка циклического импорта.

### Разворачивание скобок и запятые
Как и советуется в рекомендациях Django, если вызов некой функции или метода не вместить в одну строку, то используется `multiline hanging indentation`.

#### Примеры:

Модели:
```python
class Transaction(models.Model):
	parent_transaction = models.ForeignKey(
        to='self',
        on_delete=models.CASCADE,
        blank=True,
        null=True,
        related_name='child_transactions',
        verbose_name=_('Parent transaction'),
    )
```

ORM запросы:
```python
sent_transactions_sum = wallet.sent_transactions.exclude(
    status__in=exclude_statuses,
).filter(
    created_at__gt=last_daily_balance.day,
).aggregate(total=Sum('amount'))['total']
```

Объявление метода/функции:
```python
@classmethod
def _get_balance_for_multiple(
    cls,
    wallets: QuerySet[Wallet],
    currency: str,
    backend: ExchangeBackend,
) -> Money:
	...
```

Также, стоит обратить внимание, на необходимость ставить `trailing comma` - "конечную запятую", после последнего элемента мультилайн вызова или перечисления.

```python
class ProviderAdmin(admin.ModelAdmin):
    list_display = (
        'id',
        'name',
        'daily_transactions_count_limit',
        'wallet_daily_transactions_count_limit',
        'bank_name',
        'bank_address',
        'swift', # trailing comma
    )
```

Иногда, появляется необходимость развернуть `list comprehension`, мультилайн условие и т.д. Здесь, в первую очередь, нужно подумать о простоте и понятности кода. Если блок кода достаточно прост, понятен и выглядит лаконично, то позволяется равзернуть его. В этих случаях, операторы или ключевые слова должны находиться в начале новой строки.

List comprehension:
```python
return [
     {'amount': balance.amount, 'amount_currency': str(balance.currency)}
     for balance in balances
 ]
```

Объявление композитной переменной:
```python
amount = Decimal(
    last_daily_balance.amount.amount
    + received_transactions_sum
    - sent_transactions_sum
    - debts_sum,
)
```

Условие:
```python
if (
    provider_withdraw_transactions_count_for_today
    >= wallet.provider.daily_transactions_count_limit
):
    raise ValidationError(cls.Error.PROVIDER_DAILY_LIMIT)
```

### Именованные аргументы (kwargs)
Неименованные аргументы позволяются, только в случае, если название параметра совпадает с названием переменной, которая передаётся в неё. В любом ином кейсе, ради избежания путаницы, необходимо использовать именованные аргументы.

### Инициализаторы пакетов - __init__.py
Как и сказано в названии, `__init__.py` файлы являются инициализаторами Python пакетов, ничего кроме инициализации пакета там быть не должно. Они могут содержать только импорты, ничего больше.


## Модели
Корректное определение и написание моделей является одним из самых важных частей в разработке Django приложений, поскольку определяются сущности, вокруг которых будет реализована функциональность продукта.

При написании моделей необходимо придерживаться следующих правил:
- каждое поле модели должно иметь `verbose_name`
- скобки каждого поля должны быть развернуты - `multiline hanging indentation` + `trailing_comma`
- если поля можно группировать логические, то это стоит сделать
- поля представляющие из себя отношения к другим моделям (OneToOne, ForeignKey, ManyToMany), должны иметь `related_name`
- поля выборки должны быть реализованы через [Enum типы Django](https://docs.djangoproject.com/en/3.1/ref/models/fields/#enumeration-types)
- аргументы полей должны передаваться как именованные (kwargs)
- каждая модель должна иметь свою реализацию метода `__str__()`

### Валидация моделей
Если кастомную валидацию модели возможно реализовать через [constraints](https://docs.djangoproject.com/en/3.1/ref/models/constraints/) или [валидаторами](https://docs.djangoproject.com/en/3.1/ref/validators/) Django, то этот вариант будет лучше. В противоположном случае, стоит учесть несколько пунктов:

Кастомная валидация модели должна быть реализована в методе `clean()`, если:
- валидации немного (метод `clean()` не должен раздувать модель)
- валидация не производится по отношениям модели

В любом ином случае, валидация должна иметь свой сервис. Который может вызываться либо в `clean()` модели, либо в сериалайзере.
Также, при наличии кастомной валидации стоит не забывать вызвать метод `full_clean()`, обычно это происходит в методе `save()` модели.

### Свойства моделей (@property)
При кейсе, когда требуется наличие полей или свойств, относящиеся к инстансу модели - объекту, которые не хранятся в БД, допустимо использовать `@property`, которое позволит вычислить нужное значение в рантайме. Такие свойства допустимы, если значение вычисляется на основе полей объекта и остаются маленькими, в плане количества строк кода.

**Пример модели:**
```python
class User(AbstractBaseUser, PermissionsMixin):
    class Type(models.TextChoices):
        ADMIN = 'admin', _('Administrator')
        ACCOUNTANT = 'accountant', _('Accountant')
        CLIENT = 'client', _('Client')
        OFFICER = 'officer', _('Officer')

    email = models.EmailField(
        unique=True,
        verbose_name=_('Email'),
    )
    user_type = models.CharField(
        max_length=10,
        blank=True,
        choices=Type.choices,
        verbose_name=_('User type'),
    )
    created_at = models.DateTimeField(
        default=timezone.now,
        verbose_name=_('Created at'),
    )

    is_verified = models.BooleanField(
        default=False,
        verbose_name=_('Is verified'),
    )

    is_staff = models.BooleanField(
        default=False,
        verbose_name=_('Is staff'),
    )
    is_active = models.BooleanField(
        default=False,
        verbose_name=_('Is active'),
    )

    def __str__(self) -> str:
        return self.email

    @property
    def is_officer(self) -> bool:
        return self.groups.filter(name__in=self.Group.OFFICER_GROUPS).exists()

    @property
    def is_client(self) -> bool:
        return self.groups.filter(name__in=self.Group.CLIENT_GROUPS).exists()

    @property
    def get_client_group(self) -> Group:
        return self.groups.filter(name__in=self.Group.CLIENT_GROUPS).first()
```

## Services & selectors
Вечная проблема MVC фреймворков - "где хранить бизнес-логику?", особенно в эпоху TDD, BDD и т.д. В этом плане, Django не стал исключением. Даже учитывая то, что всеми нами любимый фреймворк реализует свой аналог этого паттерна MTV(Model, Template, View). По сути, это тот же самый MVC, где T - View, а V - Controller.

Каждый MVC фреймворк решает эту проблему по своему, зачастую существует несколько популярных решений. В случае с Django, это:
- бизнес-логика в моделях (концепция "Fat models, thin views")
- бизнес-логика в отдельных сервисах и селекторах

> Абсолютным антипаттерном и грехом, за который вас могут проклясть, является хранение бизнес-логики в контроллерах и сериалайзерах.

Давайте коротко рассмотрим каждый из наиболее популярных вариантов хранения бизнес-логики.

### Fat models, thin views
Этот вариант советуется кор-девами Django, такими как [James Bennet](https://github.com/ubernostrum), который даже писал [статьи против использования сервисного слоя](https://www.b-list.org/weblog/2020/mar/16/no-service/). В этой статье, Беннет как представитель Django и один из кор-контрибъюторов ORM, хорошо аргументирует использование тех средств, которые фреймворк даёт из коробки: модели, менеджеры и квайрисеты. Основным аргументом Джеймса является предложение не придумывать еще один слой ORM, а напротив расширять функциональность существующих интерфейсов. Но это имеет свои минусы:
- модели, менеджеры и квайрисеты становятся слишком "жирными"
- так или иначе, будут модули утилит, которые будут иметь бизнес-логику не связанную с ORM
- тестирование происходит не так удобно, как могло бы быть (например если бы бизнес-логика находилась в сервисах)

### Бизнес-логика в сервисах и селекторах
Сервисы и селекторы являются вариантом предложенным и поддерживаемым большой частью сообщества.

Сервисы и селекторы это обычные функции, в упрощенном варианте, находящиеся в модулях `services.py` и т.д.
**Сервис** - функция изменяющая состояние БД.
**Селектор** - функция читающая из БД.
Это тот самый кастомный слой ORM, о котором говорил Джеймс Беннет. Но для больших проектов он подходит лучше чем первый вариант. Тестировать удобнее и отсутствуют модули утилит, иными словами бизнес-логика становится централизованной.

### Наш подход к написанию и хранению бизнес-логики
Так как, концепция "Fat models, thin views" не подходит для больших проектов, было решено выбрать реализацию бизнес-логики через сервисы, с небольшими корректировками:
- в нашем варианте, грань между сервисами и селекторами стирается, так как сервисы и селекторы, по сути, являются композитными частями элемента бизнес-логики.
- каждый сервис хранится в соответствующем неймспейсе, на деле это класс - контейнер. Такие классы мы и называем сервисами, они имеют набор приватных методов и публичный интерфейс, который реализует требуемую бизнес-логику.
- сервис должен реализовать одну абстрактную задачу, например сервис создания транзакций. Иметь общий сервис транзакций отвечающий за создание, валидацию, подсчет и т.д. - недопустимо.
- сервис может иметь состояние, но допускается реализация сервисов и без состояния (если оно не требуется), всё зависит от кейса и требуемого результата.

Основным плюсом нашего подхода является разграниченность бизнес-логики, этого позволяют нам достичь неймспейсы - классы. В отличие от сервисов и селекторов, которые разделены по разным модулям, наш подход позволяет хранить всё нужное для реализации элемента бизнес-логики в одном контейнере. Как говорил наш отец-основатель, Tim Peters, на последней строке ["Zen of Python"](https://www.python.org/dev/peps/pep-0020/):
> "Namespaces are one honking great idea - let's do more of those!"

### Нейминг сервисов
Базовый шаблон для нейминга сервисов - `<Entity><Action>Service`, например `TransactionCreationService`, здесь в качестве сущности над которой производится действие выступает `Transaction` - транзакция, само действие - `Creation` - создание. Тем самым получается "сервис создания транзакции".

Но иногда требуется нейминг по сложнее, предположим кейс, есть сервис валидации транзакции, но поскольку валидация является обширной и сама по себе несёт много логики, решается разделить валидацию в несколько сервисов, а в корневом это маппить или как-то диспатчить. В этом случае, расширяется `Entity` и шаблон приходит к такому виду `<Type><Entity><Action>Service`, здесь `<Type>` это свойство по которому будет происходить маппинг сервисов, в итоге у нас будет `WithdrawTransactionValidationService`.

### Нейминг методов
Нейминг методов в сервисах производится по двум шаблонам, которые применяются базируясь на типе сервиса:
- если сервис реализует логику основывающуюся на имеющейся сущности, т.е. модели, то используется `<action>()`, здесь `<acton>` подразумевает короткое описание действия, например `calculate_sender_balance`.
- если сервис реализуют специфичную общую логику, но не базируется на имеющейся сущности - `<entity>_<action>()`, под этот же кейс попадают редкие, вайлдкард методы, когда имеется сервис некой сущности, но в силу специфики бизнес-логики нужен приватный метод взаимодействующий с иной сущностью, здесь стоит учесть, что этот метод должен следующему критерию: планируемый к реализации метод, по логике не может находиться в другом сервисе или быть свойством модели.

**Пример сервиса**:
```python
class TransactionCreationService:
    class Error:
        INSUFFICIENT_FUNDS_ON_WALLET = _('Insufficient funds on the wallet')

    @classmethod
    def create_withdraw(cls, data: dict) -> Transaction:
        sender = data['sender']
        state = cls._get_withdraw_transaction_initial_state(data['withdraw_method'])
        data.update(
            **cls._get_wallet_balances(sender=sender, amount=data['amount']),
            transaction_type=Transaction.Type.WITHDRAW,
            amount_currency=sender.currency,
            state=state,
        )

        with transaction.atomic():
            base_transaction = Transaction.objects.create(**data)
            fee = FeeService._get_withdraw_fee(base_transaction)
            cls._transaction_validate_wallet_balance(base_transaction, fee)
            Transaction.objects.create(
                **cls._get_wallet_balances(sender=sender, amount=fee),
                parent_transaction=base_transaction,
                sender=sender,
                recipient=cls._get_operational_wallet(sender),
                amount=fee,
                transaction_type=Transaction.Type.SERVICE,
                amount_currency=sender.currency,
                status=Transaction.Status.COMPLETED,
            )
        return base_transaction

    @classmethod
    def create_move(cls, data: dict) -> Transaction:
        data = deepcopy(data)

        sender = data['sender']
        data.update(
            **cls._get_wallet_balances(sender=sender, amount=data['amount']),
            transaction_type=Transaction.Type.MOVE,
            amount_currency=sender.currency,
        )
        return Transaction.objects.create(**data)

    @staticmethod
    def _get_operational_wallet(wallet: Wallet) -> OperationalWallet:
        return OperationalWallet.objects.get(
            provider=wallet.provider,
            currency=wallet.currency,
            wallet_type=wallet.wallet_type,
        )

    @staticmethod
    def _get_withdraw_transaction_initial_state(withdraw_method: WithdrawMethod) -> dict:
        versions = withdraw_method.version
        latest_version = max([parser.parse(version_dt) for version_dt in versions.keys()])
        return {'withdraw_method_version': str(latest_version)}

    @staticmethod
    def _get_wallet_balances(
        *args,
        sender: Wallet = None,
        recipient: Wallet = None,
        amount: Decimal,
    ) -> dict:
        balance_data = dict()
        if sender:
            sender_balance = WalletBalanceService.get_balance(sender)
            balance_data.update(
                sender_balance_before=sender_balance,
                sender_balance_before_currency=sender.currency,
                sender_balance_after=sender_balance - amount,
                sender_balance_after_currency=sender.currency,
            )
        if recipient:
            recipient_balance = WalletBalanceService.get_balance(recipient)
            balance_data.update(
                recipient_balance_before=recipient_balance,
                recipient_balance_before_currency=recipient.currency,
                recipient_balance_after=recipient_balance + amount,
                recipient_balance_after_currency=recipient.currency,
            )

        return balance_data

    @classmethod
    def _transaction_validate_wallet_balance(cls, transaction: Transaction, fee: Decimal) -> None:
        sender_wallet_balance = WalletBalanceService.get_balance(transaction.sender)
        if transaction.amount.amount + fee > sender_wallet_balance:
            raise ValueError(cls.Error.INSUFFICIENT_FUNDS_ON_WALLET)
```