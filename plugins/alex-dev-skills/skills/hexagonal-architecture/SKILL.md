---
name: hexagonal-architecture
description: Usa questa skill quando stai progettando un nuovo modulo/servizio, o rifattorizzando codice dove la logica di dominio è mischiata con framework/DB/HTTP. Enforce il pattern ports & adapters (hexagonal) per isolare la logica pura da I/O e dipendenze mutevoli. Esempi concreti in Python (Django/FastAPI) e Go.
---

# Hexagonal Architecture (Ports & Adapters)

## Principio

**La logica di dominio non deve sapere nulla di come viene servita o di dove vengono i dati.** Django, FastAPI, PostgreSQL, Celery, Redis sono **dettagli di delivery**, non il cuore del software.

Conseguenze pratiche:
- Test di dominio senza DB, senza mock framework, in millisecondi
- Cambio DB o framework = nuovo adapter, non rewrite
- Il codice descrive il **business**, non il template di un framework

## Processo

### 1. Identifica il dominio
- **Entity**: oggetti con identità (`Order`, `Backend`, `User`)
- **Value object**: immutabili, uguaglianza per valore (`Money`, `Email`)
- **Domain service**: operazioni che non appartengono a una singola entity
- **Domain event**: cose significative accadute (`OrderPlaced`, `BackendAllocated`)

Scrivili **prima** come dataclass/struct puri, senza dipendenze esterne.

### 2. Definisci le porte (interface)
- **Porta driver (inbound)**: cosa il dominio **offre** (casi d'uso) — `PlaceOrderUseCase`
- **Porta driven (outbound)**: cosa il dominio **richiede** — `OrderRepository`, `PaymentGateway`, `EmailSender`
- Le porte sono interface/Protocol **nel layer di dominio**

### 3. Implementa gli adapter
- **Adapter driver**: HTTP handler, CLI, Celery task, consumer Kafka → chiamano gli use case
- **Adapter driven**: DjangoOrderRepository, StripeGateway, SMTPEmailSender → implementano le porte

### 4. Wire-up in un composition root
- Un solo posto dove istanzi concreti e inietti (main, startup FastAPI, AppConfig Django)
- **Non** usare service locator o DI container magici per piccoli progetti — costruttori espliciti bastano

### 5. Test
- **Dominio**: test puri, senza I/O, con fake in-memory delle porte
- **Adapter**: test integrazione per ciascuno (DB reale, HTTP mock)
- **End-to-end**: pochi, critical path

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "È troppa cerimonia per un CRUD" | Forse. Ma appena il business ha regole (validazioni, workflow, pagamenti), il CRUD è già mischiato con logica. Estrai appena succede. |
| "Django ha già MTV, non serve altro" | MTV è delivery + ORM. La logica va in un layer dedicato **fuori** dai model. |
| "I Repository sono pattern anemico" | Solo se estrai solo `save/find`. Usa metodi business: `repo.active_orders_for(user)`. |
| "Le interface in Python non servono" | Servono per pensare, documentare contratti, e per typing. Usa `typing.Protocol`. |

## Esempio Python (Django + FastAPI come adapter HTTP)

### Dominio (`domain/orders.py` — zero Django)

```python
from dataclasses import dataclass
from decimal import Decimal
from typing import Protocol
from uuid import UUID, uuid4

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

@dataclass
class Order:
    id: UUID
    user_id: UUID
    total: Money
    status: str  # placed|paid|shipped|cancelled

    def mark_paid(self) -> None:
        if self.status != "placed":
            raise DomainError(f"cannot pay order in status {self.status}")
        self.status = "paid"

class OrderRepository(Protocol):
    def save(self, order: Order) -> None: ...
    def get(self, order_id: UUID) -> Order: ...
    def active_for(self, user_id: UUID) -> list[Order]: ...

class PaymentGateway(Protocol):
    def charge(self, amount: Money, token: str) -> str: ...  # returns charge_id

class DomainError(Exception): ...
```

### Use case (`domain/use_cases.py`)

```python
@dataclass
class PlaceOrderUseCase:
    orders: OrderRepository
    payments: PaymentGateway

    def execute(self, user_id: UUID, total: Money, payment_token: str) -> UUID:
        order = Order(id=uuid4(), user_id=user_id, total=total, status="placed")
        charge_id = self.payments.charge(total, payment_token)
        order.mark_paid()
        self.orders.save(order)
        return order.id
```

### Adapter driven: Django repo (`adapters/django_orders.py`)

```python
from domain.orders import Order, OrderRepository
from .models import OrderModel

class DjangoOrderRepository(OrderRepository):
    def save(self, order: Order) -> None:
        OrderModel.objects.update_or_create(
            id=order.id,
            defaults=dict(user_id=order.user_id, amount=order.total.amount,
                          currency=order.total.currency, status=order.status),
        )
    def get(self, order_id):
        m = OrderModel.objects.get(pk=order_id)
        return Order(id=m.id, user_id=m.user_id,
                     total=Money(m.amount, m.currency), status=m.status)
    def active_for(self, user_id):
        qs = OrderModel.objects.filter(user_id=user_id).exclude(status="cancelled")
        return [Order(id=m.id, user_id=m.user_id,
                      total=Money(m.amount, m.currency), status=m.status) for m in qs]
```

### Adapter driver: FastAPI route (`api/orders.py`)

```python
@router.post("/orders")
def place_order(body: PlaceOrderBody, uc: PlaceOrderUseCase = Depends(get_place_order_uc)):
    order_id = uc.execute(body.user_id, Money(body.amount, body.currency), body.token)
    return {"order_id": order_id}

# composition root (main.py o startup)
def get_place_order_uc() -> PlaceOrderUseCase:
    return PlaceOrderUseCase(
        orders=DjangoOrderRepository(),
        payments=StripeGateway(settings.STRIPE_KEY),
    )
```

### Test dominio (nessun DB, millisecondi)

```python
class InMemoryOrderRepo:
    def __init__(self): self._data = {}
    def save(self, o): self._data[o.id] = o
    def get(self, i): return self._data[i]
    def active_for(self, uid): return [o for o in self._data.values() if o.user_id == uid]

class FakePayments:
    def charge(self, amount, token): return "ch_fake_1"

def test_place_order_marks_paid():
    uc = PlaceOrderUseCase(orders=InMemoryOrderRepo(), payments=FakePayments())
    oid = uc.execute(uuid4(), Money(Decimal("10"), "eur"), "tok")
    assert uc.orders.get(oid).status == "paid"
```

## Esempio Go (net/http come adapter)

```go
// domain/backend.go
type Backend struct {
    ID     string
    Status string
}
func (b *Backend) Allocate() error {
    if b.Status != "pending" {
        return fmt.Errorf("cannot allocate backend in status %q", b.Status)
    }
    b.Status = "allocated"
    return nil
}

// domain/ports.go
type BackendRepo interface {
    Save(ctx context.Context, b *Backend) error
    Get(ctx context.Context, id string) (*Backend, error)
}
type Provisioner interface {
    Provision(ctx context.Context, id string) error
}

// domain/allocate.go
type AllocateBackend struct {
    Repo   BackendRepo
    Provis Provisioner
}
func (uc *AllocateBackend) Execute(ctx context.Context, id string) error {
    b, err := uc.Repo.Get(ctx, id)
    if err != nil { return err }
    if err := b.Allocate(); err != nil { return err }
    if err := uc.Provis.Provision(ctx, id); err != nil { return err }
    return uc.Repo.Save(ctx, b)
}

// adapters/http/allocate_handler.go
func (h *Handler) HandleAllocate(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    if err := h.uc.Execute(r.Context(), id); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest); return
    }
    w.WriteHeader(http.StatusNoContent)
}
```

## Pitfall ricorrenti

- **Model ORM usato come Entity** → il dominio dipende da Django. Estrai dataclass/struct puri.
- **Use case che ritorna oggetti ORM** → leak del detail. Ritorna dataclass o DTO.
- **Logica nei serializer/handler** → diventa intrasferibile. Spostala nell'use case.
- **Un Repository `genericRepo<T>`** → spesso anti-pattern. Meglio repo specifici con metodi business.
- **Troppe interface per cose che non cambiano mai** → pragmatismo: `EmailSender` sì, `UUIDGenerator` no.

## Exit criteria

- [ ] Il dominio (entity, VO, use case, porte) **non importa** framework/DB
- [ ] Test di dominio girano senza DB, in < 1s totale
- [ ] Ogni adapter driven implementa una porta esplicita del dominio
- [ ] Il composition root è **uno solo**, esplicito
- [ ] I handler HTTP/Celery/CLI sono thin: parse input → chiama use case → formatta output
