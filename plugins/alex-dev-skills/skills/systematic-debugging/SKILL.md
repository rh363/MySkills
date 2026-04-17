---
name: systematic-debugging
description: Usa questa skill quando devi investigare un bug non banale — comportamento anomalo in produzione, test flaky, errori intermittenti, problemi distribuiti (Celery, Kubernetes, Traefik, code async). Enforce un processo in 5 step (reproduce → localize → reduce → fix → guard) che evita guess-and-check e fix cosmetici.
---

# Systematic Debugging

## Principio

**Debug è investigazione, non tentativi.** Un bug che "sparisce" senza capire perché, tornerà. Un fix che funziona "non so bene come" è debito tecnico travestito.

Il metodo in 5 step forza disciplina: non salti a fixare prima di aver riprodotto, non riduci l'ambito troppo presto, non chiudi senza una guardia contro la regressione.

## Processo

### 1. REPRODUCE — rendilo deterministico
- Un bug non riproducibile **non è debuggabile**. Lavora prima su come riprodurlo.
- Isola input, stato, timing, concorrenza. Annota i valori esatti.
- Se è intermittente → trova l'invariante che lo scatena (N richieste, race con task X, cache fredda, TZ al confine).
- Scrivi il reproducer come test failing (vedi `tdd-prove-it`) **appena possibile**.

### 2. LOCALIZE — dove sta il problema
- Usa bisect: dimezza lo spazio di ricerca finché non hai il punto preciso.
  - `git bisect` per regressioni temporali
  - Stub/mock progressivi per dipendenze
  - Log aggiuntivi a diverse profondità
- **Non cambiare codice** in questa fase. Solo osservare.
- Fermati quando hai il **frame singolo** (funzione/riga) in cui l'invariante si rompe.

### 3. REDUCE — minimal reproducer
- Elimina tutto ciò che non è necessario a scatenare il bug
- Un buon reproducer è: pochi file, pochi input, tempo di esecuzione < 10s
- Se il reproducer è grande, il fix sarà approssimativo

### 4. FIX — causa radice, non sintomo
- Chiediti: **perché** succede? (Poi di nuovo: **perché**? 5 volte almeno)
- Fix della causa, non del sintomo. Esempi:
  - Sintomo: "il retry raddoppia il pagamento" → Fix sintomo: aggiungi flag. Fix causa: idempotency key lato DB.
  - Sintomo: "campo null crasha il render" → Fix sintomo: `?? ''`. Fix causa: la query a monte ritorna null inatteso.
- Il fix deve rendere verde il reproducer della fase 1.

### 5. GUARD — regressione impossibile
- Il test del reproducer rimane nel repo **forever**
- Se il bug era concorrente → aggiungi un test di stress/fuzz
- Se era una condizione di boundary → aggiungi property test
- Documenta nel commit message il **perché** (non solo il cosa)

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "Non riesco a riprodurlo localmente, fixo a naso" | Stai indovinando. Il "fix" potrebbe peggiorare. Investi in un reproducer. |
| "È un bug nel framework/libreria" | Forse. Dimostralo con un reproducer minimale prima di aprire issue. |
| "Riavviando si risolve" | Hai nascosto il problema, non risolto. Tornerà. |
| "Aggiungo try/except e vado avanti" | Hai trasformato un bug visibile in uno silenzioso. Peggio. |

## Esempio Python — Celery task intermittente (ecs2)

```python
# Sintomo: a volte il backend resta allocato dopo delete
# Step 1-REPRODUCE: stress con delete concorrenti
def test_concurrent_delete_releases_backend(db):
    backend = BackendFactory(status="allocated")
    with ThreadPoolExecutor(max_workers=10) as ex:
        futures = [ex.submit(delete_backend, backend.id) for _ in range(10)]
        [f.result() for f in futures]
    backend.refresh_from_db()
    assert backend.status == "released"  # FALLISCE: resta "allocated" nel 30% dei run
```

```python
# Step 2-LOCALIZE: log su transition. Vedo che release_backend viene chiamato,
#   ma lo SELECT ... FOR UPDATE non acquisisce il lock (read uncommitted).
# Step 3-REDUCE: reproducer a 2 task concorrenti è sufficiente.
# Step 4-FIX causa: usare SELECT FOR UPDATE SKIP LOCKED + isolation level SERIALIZABLE
#   sul path di release, non patch a livello di retry.
# Step 5-GUARD: il test di stress rimane in CI con seed fisso.
```

## Esempio Go — race su shared map

```go
// Sintomo: panic "concurrent map writes" in produzione
// REPRODUCE: go test -race con N goroutine
func TestConnCache_ConcurrentAccess(t *testing.T) {
    cache := NewConnCache()
    var wg sync.WaitGroup
    for i := 0; i < 100; i++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            cache.Set(fmt.Sprintf("k%d", id), id)
            _ = cache.Get(fmt.Sprintf("k%d", id))
        }(i)
    }
    wg.Wait()
    // -race FALLISCE: data race su map interna
}
// FIX: sync.RWMutex o sync.Map a seconda del pattern read/write.
// GUARD: il test con -race rimane in CI.
```

## Esempio TypeScript — bug timing in SvelteKit hook

```ts
// Sintomo: sessione a volte valida, a volte no, su stesso token
// REPRODUCE: test che chiama hook 2 volte in parallelo
it('handle sets locals.user exactly once per request', async () => {
  const event = mockEvent({ cookies: { session: validToken } });
  const [a, b] = await Promise.all([handle({event, resolve}), handle({event, resolve})]);
  expect(event.locals.user).toBeDefined();
  // FALLISCE nel 20% dei run perché handle muta shared state
});
// LOCALIZE: state singleton a livello modulo.
// FIX: locals.user deriva per-request, non singleton.
```

## Quando arrenderti e chiedere aiuto

- Hai speso > 2h senza nemmeno riprodurre → chiedi a un collega di guardare insieme
- Il bug è nel kernel/hypervisor/infra → escalation, non heroics
- Hai sospetti su una libreria → isolalo in un repo minimo prima di aprire issue upstream

## Exit criteria

- [ ] Hai un reproducer deterministico (test o script) committato
- [ ] Hai identificato la **causa radice**, non solo il sintomo
- [ ] Il fix fa passare il reproducer, non lo aggira
- [ ] Hai una guardia (test, assertion, check runtime) contro la regressione
- [ ] Il commit message spiega il **perché** del bug, non solo il **cosa** del fix
