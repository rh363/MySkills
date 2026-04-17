---
name: tdd-prove-it
description: Usa questa skill quando stai per implementare una feature, fixare un bug, o modificare comportamento esistente. Enforce il pattern Prove-It — riproduci il problema con un test che fallisce PRIMA di scrivere il fix. Applicabile a Python (pytest), Go (testing/testify), TypeScript (vitest/jest).
---

# TDD Prove-It

## Principio

**Nessun fix senza un test che prima fallisce.**

Il test che fallisce è la prova che:
1. Hai capito il problema
2. Il fix effettivamente risolve qualcosa
3. Il bug non tornerà silenziosamente

Scrivere solo il fix ("tanto vedo che funziona") significa:
- Non hai verificato di aver riprodotto il bug reale
- Non hai una guardia contro regressioni
- Il prossimo refactor romperà tutto di nuovo

## Processo (RED → GREEN → REFACTOR)

### 1. RED — scrivi un test che fallisce
- Il test deve riprodurre **esattamente** il comportamento che non va
- Esegui il test e **vedi il fallimento** (non assumerlo)
- Leggi il messaggio di errore: è quello che ti aspettavi?

### 2. GREEN — il fix minimo che fa passare il test
- Scrivi il codice più piccolo possibile che rende verde il test
- Non aggiungere funzionalità "già che ci sono"
- Esegui il test e **vedi il verde** (non assumerlo)

### 3. REFACTOR — pulisci mantenendo verde
- Ora che hai la safety net del test, puoi rifattorizzare
- Esegui i test dopo ogni modifica significativa
- Se diventa rosso, non procedere: capisci perché

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "È un fix banale, non serve un test" | Se è banale, il test è veloce. Scrivilo. |
| "Aggiungo i test dopo" | Non li aggiungerai. E se lo fai, saranno scritti per passare, non per provare. |
| "Il codice è già testato manualmente" | Il test manuale non verrà rieseguito al prossimo refactor. |
| "Non so come testarlo" | Allora non sai come sapere se funziona. Ferma, pensa al design. |

## Esempio Python (pytest) — bug Celery idempotency

```python
# test_payment_task.py - RED
def test_process_payment_is_idempotent(db, payment_factory):
    payment = payment_factory(amount=100, status="pending")
    # Chiamiamo 2 volte la stessa task (simula retry Celery)
    process_payment.apply(args=[payment.id])
    process_payment.apply(args=[payment.id])
    payment.refresh_from_db()
    # Deve essere stato processato UNA volta sola
    assert payment.status == "processed"
    assert ExternalCharge.objects.filter(payment=payment).count() == 1
```

Esegui: `pytest test_payment_task.py::test_process_payment_is_idempotent`.
Deve fallire (o perché crea 2 charges, o perché crasha). Solo ora scrivi il fix.

## Esempio Go (testify) — reproducer concorrenza

```go
// worker_test.go - RED
func TestAcquireLock_PreventsDoubleAllocation(t *testing.T) {
    store := newTestStore(t)
    var wg sync.WaitGroup
    acquired := atomic.Int32{}

    for i := 0; i < 10; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            if ok, _ := store.AcquireBackendLock(ctx, "backend-1"); ok {
                acquired.Add(1)
            }
        }()
    }
    wg.Wait()
    // Solo UN goroutine deve ottenere il lock
    assert.Equal(t, int32(1), acquired.Load())
}
```

## Esempio TypeScript (vitest) — validazione input

```ts
// auth.test.ts - RED
import { describe, it, expect } from 'vitest';
import { validateCallback } from './auth';

describe('validateCallback', () => {
  it('rejects state param not matching stored state', () => {
    const result = validateCallback({
      state: 'attacker-state',
      storedState: 'original-state',
    });
    expect(result.ok).toBe(false);
    expect(result.error).toBe('STATE_MISMATCH');
  });
});
```

## Quando il test è "difficile" da scrivere

Se ti trovi a dire "è impossibile testare questo pezzo", il problema è nel **design**, non nel test:

- Logica mischiata con I/O → estrai la logica pura
- Dipendenze hard-coded → inietta (DI, parametri)
- Stato globale → incapsulalo in un oggetto

La difficoltà di testare è un code smell, non una scusa.

## Exit criteria

- [ ] Hai scritto un test che **fallisce** riproducendo il comportamento da fixare
- [ ] Hai eseguito il test e visto il fallimento (non assunto)
- [ ] Il fix fa passare quel test specifico
- [ ] Gli altri test non si sono rotti
- [ ] Il test è committato insieme al fix, non dopo
