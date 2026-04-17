---
name: celery-idempotency
description: Usa questa skill quando stai scrivendo o modificando una task Celery che ha side effect (DB write, chiamata esterna, invio email, allocazione risorsa, pagamento). Enforce idempotency-by-design così che retry automatici (worker crash, ack_late, visibility timeout) non producano duplicati o stati corrotti. Pattern per Django + PostgreSQL.
---

# Celery Idempotency

## Principio

**Una task Celery può essere eseguita più di una volta.** Sempre. Retry su eccezione, worker ucciso prima dell'ACK, visibility timeout su broker, duplicazione da producer flaky: assumi il doppio run come normale.

Se la tua task non è idempotente, non è "a volte buggata" — è **rotta per design**.

## Processo

### 1. Classifica la task
- **Pure (nessun side effect)** → idempotente per costruzione, nessun lavoro extra.
- **Side effect locale (DB only)** → usa unique constraint + check-then-act transazionale.
- **Side effect esterno (API, email, pagamento)** → idempotency key lato chiamante + log del risultato.

### 2. Scegli il meccanismo di idempotenza
| Tecnica | Quando |
|---|---|
| **Unique constraint DB** | Quando l'oggetto creato ha un identificatore naturale (order_id, task_id) |
| **Idempotency key** | API esterne (Stripe, provider cloud, SMTP con Message-ID) |
| **State machine + guard** | La task fa transition (e.g. `pending → processing → done`), usa `UPDATE ... WHERE status = 'pending'` |
| **Distributed lock** | Operazioni che non mappano a una singola riga (rebuild cache, cleanup) — usa Redis SET NX PX o advisory lock PG |

### 3. Transazionalità
- **Il side effect esterno e il DB write devono essere coordinati.**
- Pattern outbox: scrivi l'intento nel DB dentro la transazione, un worker separato lo consuma.
- **Evita**: chiamata esterna dentro `atomic()` (rollback non la cancella), o DB write dopo la chiamata esterna (se crasha, hai charge ma no record).

### 4. Configurazione Celery sicura
- `acks_late=True` + `task_reject_on_worker_lost=True` → retry se il worker muore, ma richiede idempotenza.
- `max_retries` esplicito, `retry_backoff=True` con jitter.
- `autoretry_for` con lista specifica di eccezioni (no `Exception` nudo).

### 5. Osservabilità
- Log `task_id` + `idempotency_key` ad ogni step
- Metrica sui duplicate-detected (dovrebbe essere > 0 in produzione se la task è giusta)

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "I retry sono rari" | In produzione, su volumi reali, succede. Una volta sola basta. |
| "Il mio broker è affidabile" | Il problema non è il broker, è il **worker** che può morire dopo il side effect. |
| "Aggiungo un check all'inizio" | Check-then-act senza lock è race condition classica. Serve atomicità. |
| "Uso `acks_early`" | Perdi task se il worker muore. Scelta valida solo per task best-effort (notifiche non critiche). |

## Esempio 1 — Pagamento esterno con idempotency key (Stripe-like)

```python
# models.py
class PaymentAttempt(models.Model):
    idempotency_key = models.CharField(max_length=64, unique=True)
    payment_id = models.ForeignKey(Payment, on_delete=models.PROTECT)
    external_charge_id = models.CharField(max_length=128, null=True)
    status = models.CharField(max_length=16)  # pending|succeeded|failed
    created_at = models.DateTimeField(auto_now_add=True)

# tasks.py
@shared_task(bind=True, autoretry_for=(ProviderTimeout, ProviderServerError),
             retry_backoff=True, max_retries=5, acks_late=True)
def process_payment(self, payment_id: int):
    payment = Payment.objects.get(pk=payment_id)
    # Idempotency key deterministica: payment_id + attempt_number di Celery
    # (ma stabile tra retry della STESSA task → self.request.id)
    key = f"pay-{payment_id}-{self.request.id}"

    attempt, created = PaymentAttempt.objects.get_or_create(
        idempotency_key=key,
        defaults={"payment_id": payment, "status": "pending"},
    )
    if attempt.status == "succeeded":
        return attempt.external_charge_id  # già fatto, no-op

    # Stripe accetta idempotency key → se chiamiamo due volte con stessa key,
    #   ritorna lo STESSO charge senza duplicare
    charge = stripe.Charge.create(
        amount=payment.amount_cents,
        currency="eur",
        source=payment.source_token,
        idempotency_key=key,
    )
    # UPDATE atomico
    PaymentAttempt.objects.filter(pk=attempt.pk, status="pending").update(
        external_charge_id=charge.id, status="succeeded",
    )
    return charge.id
```

## Esempio 2 — State machine con guard SQL

```python
@shared_task(acks_late=True)
def allocate_backend(order_id: int):
    # Transizione pending → allocating in un solo UPDATE, solo se era pending.
    # Se un altro worker c'è già riuscito, updated == 0 e abortiamo.
    updated = Order.objects.filter(pk=order_id, status="pending").update(
        status="allocating", allocated_at=timezone.now(),
    )
    if updated == 0:
        logger.info("order %s already handled by another worker", order_id)
        return
    try:
        backend_id = provision_backend()
        Order.objects.filter(pk=order_id).update(
            status="allocated", backend_id=backend_id,
        )
    except Exception:
        # Rollback esplicito della transizione di stato
        Order.objects.filter(pk=order_id).update(status="pending")
        raise
```

## Esempio 3 — Distributed lock per operazione non-row-scoped

```python
from redis.exceptions import LockError

@shared_task(bind=True)
def rebuild_search_index(self):
    lock = redis_client.lock("lock:rebuild_search_index", timeout=600, blocking=False)
    if not lock.acquire():
        logger.info("rebuild already running, skipping")
        return
    try:
        # operazione non-idempotente ma protetta da lock
        do_rebuild()
    finally:
        try:
            lock.release()
        except LockError:
            pass  # TTL già scaduto, fine
```

## Pitfall ricorrenti

- **Chiamata esterna dentro `transaction.atomic()`** → il rollback non cancella la chiamata, ma il record sì. Separa.
- **`get_or_create` senza unique constraint** → race: due worker creano due record. Mettere `unique=True` sul campo.
- **`self.retry()` senza `max_retries`** → infinite retry su bug permanente. Sempre esplicitare.
- **`signal post_save` che triggera la task** → se la task modifica la row, loop. Usa `update_fields` o disabilita signal.

## Exit criteria

- [ ] La task ha un meccanismo di idempotenza **esplicito** (key, unique, state guard, o lock)
- [ ] Test di integrazione che esegue la task **due volte di fila** e verifica 1 solo side effect
- [ ] `acks_late=True` se il side effect è critico
- [ ] `autoretry_for` elenca eccezioni specifiche, non `Exception`
- [ ] Se chiamata esterna: idempotency key passata al provider quando supportata
- [ ] Log includono `task_id` e `idempotency_key`
