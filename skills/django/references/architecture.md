# Архитектура Django-приложения

Django MTV расширяется сервисным слоем: бизнес-логика не во views и не в моделях.

## Слои и границы

| Слой      | Знает про                      | Не знает про           | Тестируется         |
| --------- | ------------------------------ | ---------------------- | ------------------- |
| Views     | HTTP, формы, сервисы           | ORM-логику, транзакции | BDD                 |
| Services  | домен, репозитории, транзакции | `request`/`response`   | unit (моки)         |
| Selectors | запросы на чтение              | мутации                | через сервис/BDD    |
| Models    | структуру данных               | бизнес-процессы        | не тестируются unit |
| Forms     | валидацию данных               | персистентность        | BDD/ручная          |

## Сервисы

```python
from typing import Protocol
from django.db import transaction

class UserRepository(Protocol):
    def filter(self, **kwargs) -> "QuerySet": ...
    def create_user(self, email: str, password: str) -> "User": ...

class UserService:
    def __init__(
        self,
        user_repo: UserRepository,                    # 🔴 явно, без fallback
        email_service: "EmailService | None" = None,  # 🟡 вторичная — fallback ок
    ) -> None:
        self.user_repo = user_repo
        self.email_service = email_service or EmailService()

    def create_user(self, email: str, password: str) -> "User":
        with transaction.atomic():
            if self.user_repo.filter(email=email).exists():
                raise UserAlreadyExistsError(email)
            user = self.user_repo.create_user(email, password)
            transaction.on_commit(lambda: self.email_service.send_welcome(user.id))
            return user
```

Правила:

1. **Репозитории — только явно через `__init__`, fallback запрещён.**
   `self.user_repo = user_repo or User.objects` скрывает зависимость и ломает тесты.
2. **Вторичные зависимости** (сервисы, утилиты, клиенты) могут иметь безопасный
   fallback.
3. **Транзакции** — внутри сервиса (`transaction.atomic()`); побочные эффекты —
   в `transaction.on_commit()`.
4. **Сервис не знает про `request`.** Принимает данные (`user_id`), не объект запроса.
5. **Типизация** всех аргументов и возвращаемых значений.
6. **Всё сохранение — через репозиторий.** Прямой `obj.save()` в сервисе запрещён:
   иначе unit-тест требует БД.

## Selectors

Selector — функция чтения без мутаций. Выносится, если запрос:

1. сложный (JOIN, аннотации, агрегации, несколько `select_related`/`prefetch_related`), **или**
2. переиспользуется в двух и более местах.

```python
# selectors.py
from django.db.models import QuerySet, Prefetch

def get_orders_with_items(user_id: int) -> QuerySet[Order]:
    """Заказы пользователя со связанными данными (без N+1)."""
    return (
        Order.objects.filter(user_id=user_id)
        .select_related("customer")
        .prefetch_related(
            Prefetch("items", queryset=OrderItem.objects.select_related("product"))
        )
    )
```

Простые `filter(status=...)`/`get(pk=...)` остаются в сервисе или менеджере.

## Модели

```python
class Order(models.Model):
    customer = models.ForeignKey(
        "Customer", on_delete=models.PROTECT, related_name="orders"
    )
    status = models.CharField(max_length=20, choices=OrderStatus.choices)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]
        verbose_name = "заказ"
        indexes = [models.Index(fields=["customer", "-created_at"])]

    def __str__(self) -> str:
        return f"Order #{self.pk}"
```

- Явные `on_delete` и `related_name` у FK/M2M.
- Индексы на часто фильтруемых/сортируемых полях.
- Абстрактная база для общих полей (`created_at`/`updated_at`).
- Модель — данные и простые свойства; бизнес-логика — в сервисе.

## Forms

```python
class SignupForm(forms.ModelForm):
    class Meta:
        model = User
        fields = ["email", "username"]

    def clean_email(self) -> str:
        email = self.cleaned_data["email"].lower()
        if User.objects.filter(email__iexact=email).exists():
            raise forms.ValidationError("Email уже занят")
        return email
```

- Вся валидация — в форме (`clean_<field>`, `clean`), не во view.
- `fields` перечислять явно (никакого `"__all__"` — mass assignment).
- `ModelForm`, когда форма соответствует модели.

## Views: FBV и CBV

```python
# FBV — простой сценарий
def order_list(request: HttpRequest) -> HttpResponse:
    """Список заказов пользователя с пагинацией."""
    page_obj = Paginator(get_orders_with_items(request.user.id), 20).get_page(request.GET.get("page"))
    return render(request, "orders/list.html", {"page_obj": page_obj})

# CBV — CRUD-экран
class OrderCreateView(LoginRequiredMixin, CreateView):
    model = Order
    form_class = OrderForm
    success_url = reverse_lazy("orders:list")

    def form_valid(self, form: OrderForm) -> HttpResponse:
        OrderService(order_repo=Order.objects).create(
            user_id=self.request.user.id, **form.cleaned_data
        )
        return redirect(self.get_success_url())
```

- FBV — простые сценарии; CBV — CRUD-экраны. Логика — в сервисе в обоих случаях.
- `get_queryset()` вместо статического `queryset`, когда нужна динамика/права.
- Права: `LoginRequiredMixin`/`@login_required`; не отдавать чужие объекты.
- URL — через `path()` и `reverse()`, без хардкода.

## Структура проекта

```
project/
├── manage.py
├── core/
│   ├── settings/            # base / dev / prod
│   └── urls.py
├── apps/
│   └── orders/
│       ├── models.py
│       ├── views.py
│       ├── services.py
│       ├── selectors.py
│       ├── forms.py
│       ├── middleware.py
│       ├── exceptions.py
│       └── tests/
└── .env
```

## Антипаттерны

| ❌                                 | ✅                                |
| ---------------------------------- | --------------------------------- |
| Бизнес-логика во view              | Сервис                            |
| Бизнес-логика в модели             | Сервис                            |
| `self.repo = repo or User.objects` | Явная передача репозитория        |
| `user.save()` в сервисе            | `repo.create_user(...)`           |
| Толстый `form_valid` с логикой     | Вызов сервиса                     |
| `fields = "__all__"`               | Явный список                      |
| `redirect("/users/")`              | `redirect(reverse("users:list"))` |

## Чек-лист

- [ ] Views тонкие, логика в сервисах.
- [ ] Репозитории внедрены явно, без fallback.
- [ ] Сервис не знает про `request`.
- [ ] Сохранение — через репозиторий, не `save()` в сервисе.
- [ ] Сложные/переиспользуемые запросы — в selectors.
- [ ] Валидация — в формах, `fields` без `"__all__"`.
- [ ] Права проверяются, URL через `reverse()`.
