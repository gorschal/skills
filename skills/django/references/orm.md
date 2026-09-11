# ORM и производительность

Главные враги Django-приложения: N+1-запросы, отсутствие пагинации, операции
в Python вместо БД.

## N+1 и eager loading

```python
# ❌ N+1: запрос на каждую итерацию
for order in Order.objects.all():
    print(order.customer.name)

# ✅ select_related — FK / OneToOne (JOIN)
orders = Order.objects.select_related("customer")

# ✅ prefetch_related — M2M / reverse FK (отдельный запрос + склейка)
orders = Order.objects.prefetch_related("items")

# ✅ вложенные связи
orders = Order.objects.prefetch_related(
    "items__product",
    "customer__profile",
)

# ✅ Prefetch с фильтром/сортировкой
from django.db.models import Prefetch
orders = Order.objects.prefetch_related(
    Prefetch("items", queryset=OrderItem.objects.select_related("product").order_by("-created_at"))
)
```

| Метод              | Когда                                         |
| ------------------ | --------------------------------------------- |
| `select_related`   | FK, OneToOne (JOIN)                           |
| `prefetch_related` | M2M, reverse FK, `items__product`             |
| `Prefetch`         | когда нужно фильтровать/сортировать связанные |

Правило: **никогда не обращаться к связанному полю в цикле без предзагрузки.**

## Частичная загрузка

```python
User.objects.only("id", "email")
User.objects.defer("bio", "avatar")
```

Применять осознанно: `defer` откладывает поля, но обращение к ним даст лишний запрос.

## Агрегации и выражения

```python
from django.db.models import Count, Avg, Sum, F, Q

# одна строка агрегатов
Product.objects.aggregate(avg_price=Avg("price"), total=Sum("stock"))

# вычисляемое поле
categories = Category.objects.annotate(products_count=Count("products")).filter(products_count__gt=0)

# операция на стороне БД
Product.objects.update(price=F("price") * 1.1)

# сложные условия
Product.objects.filter(Q(price__lt=100) | Q(stock__gt=50), is_active=True)
```

## Кастомные QuerySet / Manager

Часто используемые фильтры — в chainable-методах.

```python
class OrderQuerySet(models.QuerySet):
    def paid(self) -> "OrderQuerySet":
        return self.filter(status=OrderStatus.PAID)

    def recent(self) -> "OrderQuerySet":
        return self.filter(created_at__gte=timezone.now() - timedelta(days=7))

class Order(models.Model):
    objects = OrderQuerySet.as_manager()
```

## Массовые операции

```python
Product.objects.bulk_create(items, batch_size=1000)

Product.objects.filter(category=old).update(category=new)

products = list(Product.objects.filter(is_active=True))
for p in products:
    p.stock += 10
Product.objects.bulk_update(products, ["stock"], batch_size=1000)
```

Не вызывать `save()` в цикле — это N запросов.

## Пагинация

Списочные эндпоинты не возвращают `.all()` без пагинации.

```python
from django.core.paginator import Paginator

paginator = Paginator(Order.objects.select_related("customer"), 20)
page_obj = paginator.get_page(request.GET.get("page"))
return render(request, "orders/list.html", {"page_obj": page_obj})
```

## Работа со временем

```python
USE_TZ = True
TIME_ZONE = "UTC"

from django.utils import timezone
now = timezone.now()          # ✅
datetime.now()                # ❌ без tz
```

Все даты — в UTC; отображение — в локальной зоне на уровне представления.

## Индексы

- Индекс на поля частых `filter`/`order_by`/`WHERE`.
- Составные индексы — под конкретный паттерн запроса (`["customer", "-created_at"]`).
- Индекс на FK создаётся автоматически; проверяйте `db_index` на «горячих» полях.
- Детали и планы запросов — в навыке `postgres`.

## Антипаттерны

| ❌                              | ✅                                  |
| ------------------------------- | ----------------------------------- |
| Доступ к related в цикле        | `select_related`/`prefetch_related` |
| `.all()` без пагинации          | `Paginator`                         |
| `save()` в цикле                | `bulk_update`                       |
| Агрегация в Python (`sum(...)`) | `aggregate`/`annotate`              |
| `F()` заменён чтением-записью   | `F()` для операций на БД            |
| `datetime.now()`                | `timezone.now()`                    |
| N+1 в шаблоне                   | предзагрузка в selector/view        |

## Чек-лист

- [ ] Нет N+1; связанные объекты предзагружены.
- [ ] Списочные эндпоинты пагинированы.
- [ ] Массовые операции — `bulk_*`/`update`, не цикл `save()`.
- [ ] Агрегации — на стороне БД.
- [ ] Время — `timezone.now()`, `USE_TZ=True`.
- [ ] Индексы под частые запросы.
