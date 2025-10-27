Resolver estas cosas sin depender de frameworks:

1. **Registrar movimientos** (gasto, ingreso, transferencia) con reglas de categorización.
2. **Modelar deudas** (tarjetas y préstamos), su saldo, ciclos y pagos con split interés/principal/comisiones.
3. **Evitar duplicados** por idempotencia.
4. **Resumir el estado** por periodo: ingresos, gastos, flujo neto, saldos por cuenta y por deuda.
5. **Aplicar reglas** simples y corregibles.
6. **Mantener límites e invariantes**: monedas, fechas, pertenencia a hogar, signos, etc.

# Objetos del dominio y atributos

## Value Objects (inmutables, comparables)

* **Money**
  `amount`, `currency`
  Reglas: no mezclar monedas; operaciones básicas; redondeo bancario.
* **Currency**
  `code` ∈ {COP, USD}; `scale` (decimales).
* **TimeSpan**
  `start`, `end` para agrupar por periodos.
* **Percent**
  `value` en [0, 1]; util para tasas y confianza de reglas.
* **Installments**
  `count`, `plan` (1 si contado), solo para compras con TC.
* **FxRate** (opcional v1)
  `from`, `to`, `rate`, `as_of`.

## Entidades núcleo

* **Household**
  `id`, `name`, `base_currency`, `created_at`, `status`.
* **Actor**
  `id`, `household_id`, `display_name`, `status`.
* **Account**
  `id`, `household_id`, `name`, `type` ∈ {bank, wallet, credit_card, cash}, `currency`, `metadata`, `status`.
  Invariantes: una cuenta tiene una sola moneda; una TC es `type=credit_card`.
* **Category**
  `id`, `household_id?` (puede ser global), `name`, `parent_id?`, `path`.
* **Transaction**
  `id`, `household_id`, `actor_id`, `account_id`, `type` ∈ {expense, income, transfer, adjustment},
  `money: Money`, `tx_ts`, `category_id?`, `subcategory_id?`, `merchant?`, `description?`, `tags[]`, `confidence: Percent`, `source`, `external_refs`.
  Invariantes: `amount>0`; para `transfer` se usa entidad dedicada.
* **Transfer**
  `id`, `household_id`, `actor_id`, `from_account_id`, `to_account_id`,
  `money: Money` (moneda idéntica en ambas cuentas), `fee_money?: Money`, `tx_ts`, `note?`.
  Invariantes: `from ≠ to`; transferencias no cuentan como gasto/ingreso; la **comisión** sí es gasto.
* **Debt**
  `id`, `household_id`, `type` ∈ {credit_card, loan, other}, `lender`, `currency`,
  `rate_type` ∈ {EA, NMV, NA}, `rate_value: Percent?`, `cutoff_day?`, `due_day?`, `amortization` ∈ {Revolving, PRICE, German, American, Custom}, `status`.
  Para TC: `type=credit_card` + `cutoff_day` y `due_day`.
* **DebtMovement**
  `id`, `debt_id`, `kind` ∈ {purchase, payment, interest, fee, adjustment},
  `money: Money`, `tx_ts`, `related_tx_id?` (p. ej. enlazar compra con transacción), `meta?`.
  Invariantes: `purchase` aumenta saldo; `payment` lo reduce; `interest/fee` cuentan como gasto.
* **Rule**
  `id`, `scope` ∈ {actor, household, global},
  `match` (merchant_contains, description_regex, account_id, amount_range, tag_present),
  `action` (category_id, subcategory_id, tags_add[]), `enabled: bool`, `priority: int`.
* **IdempotencyKey**
  `household_id`, `key`, `created_at`.
  Invariante: única por hogar.
* **Receipt** (v2)
  `tx_id`, `path`, `sha256`, `content_type`.

## Servicios (casos de uso, puros)

* **record_expense(cmd)**
  Crea Transaction expense; aplica reglas; si la cuenta es TC, crea DebtMovement purchase.
* **record_income(cmd)**
  Crea Transaction income.
* **record_transfer(cmd)**
  Crea Transfer, y si hay comisión crea Transaction expense para la fee.
* **create_debt(cmd)**
  Define Debt (TC o préstamo).
* **pay_debt(cmd)**
  Registra DebtMovement payment; registra Transaction expense por `interest` y `fees` del split; no duplica gasto.
* **compute_cc_cycle(debt_id, month)**
  Calcula saldo de ciclo, pago mínimo, pago total, fechas.
* **summarize_period(span, household_id)**
  Agrega ingresos, gastos, flujo neto, saldos por cuenta, intereses pagados, top categorías.
* **apply_rules(tx_like)**
  Devuelve categoría, subcategoría y confianza según Rule set.

## Interfaces (ports, sin implementación)

* **TransactionRepo**: `add`, `get`, `list_by_period`, `list_by_account`, etc.
* **TransferRepo**: `add`, `list_by_period`.
* **DebtRepo**: `add`, `get`, `list`, `add_movement`, `list_movements`, `balance_at(date)`, `cc_cycle(date)`.
* **RuleRepo**: `list_active_by_scope`, `add`, `disable`.
* **CategoryRepo**: `get_by_name_path`, `ensure`.
* **IdempotencyStore**: `exists(household, key)`, `save(household, key)`.
* **UnitOfWork**: contexto transaccional `__aenter__/__aexit__`, `commit`, acceso a repos.

## Eventos de dominio (opcional pero útil)

* **ExpenseRecorded**
* **DebtPurchaseRegistered**
* **DebtPaymentApplied**
* **TransferRecorded**
* **RulesAppliedToTransaction**


# Invariantes y políticas clave

* **Moneda consistente**: no sumar Money con currencies distintas; Transfer exige misma moneda en ambas cuentas.
* **Household scoping**: toda entidad referencia un `household_id` y no cruza hogares.
* **Idempotencia**: ningún caso de uso procesa un `idempotency_key` ya visto.
* **Tipos de transacción**: `expense` y `income` son para flujo contra cuentas; **pagos de deuda no son gasto** salvo interés/fee.
* **Tarjeta de crédito**: compras generan `purchase` en Debt; pagos generan `payment` y gastos por interés/fee si vienen en split.
* **Fechas y zonas horarias**: guardar `tx_ts` en UTC con offset original registrado como metadato.
* **Rounding**: redondeo bancario a `scale` de la moneda al final de cada operación.
* **Categorías**: las reglas no sobreescriben una categoría provista explícitamente por el usuario, salvo política contraria.
* **Transfer matching**: una transferencia nunca produce ingreso o gasto duplicado; su fee sí es gasto de “comisiones”.


# Comandos de entrada (DTO del dominio, no de la API)

Sin código, pero conceptual:

* **RecordExpenseCmd**: household_id, actor_id, account_id, money, tx_ts, category_id?, subcategory_id?, merchant?, description?, tags[], idempotency_key, installments?
* **RecordIncomeCmd**: idem para income.
* **RecordTransferCmd**: household_id, actor_id, from_account_id, to_account_id, money, fee_money?, tx_ts, note?, idempotency_key.
* **CreateDebtCmd**: debt params.
* **PayDebtCmd**: household_id, debt_id, money, tx_ts, split{interest, principal, fees}, paid_from_account_id, idempotency_key.
* **SummarizePeriodQuery**: household_id, span.


# Plan de pruebas: qué testear y por qué

## A. Pruebas de Value Objects

1. **Money suma/resta misma moneda**; lanza error si moneda difiere; respeta redondeo.
2. **Percent límites** 0 a 1; rechaza fuera de rango.
3. **Installments** no acepta 0 o negativo.
4. **Currency scale** aplica correctamente decimales para COP y USD.

## B. Pruebas de entidades

5. **Transaction creación válida**: tipos válidos, amount positivo, household obligatorio.
6. **Transfer invariantes**: from≠to, misma moneda, fee opcional como `Money`.
7. **DebtMovement reglas**: purchase suma, payment resta, interest/fee marcan gasto atribuible a deuda.
8. **Debt tarjeta requiere cutoff y due day**; préstamo requiere amortización válida.
9. **Category path** se construye correctamente padre/hijo.

## C. Casos de uso básicos

10. **record_expense con regla**: aplica categoría y asigna `confidence`; si usuario trae categoría, la prioriza.
11. **record_expense con TC**: crea Transaction y además DebtMovement purchase vinculado.
12. **record_income**: crea ingreso sin categoría de gasto.
13. **record_transfer sin fee**: no crea gasto; impacta saldos lógicos solo en reporte.
14. **record_transfer con fee**: crea gasto de comisión aparte.
15. **idempotencia en record_expense**: segunda llamada con misma clave falla controladamente.
16. **pay_debt split completo**: reduce saldo de deuda por principal; registra gastos por interest y fees; no duplica.
17. **pay_debt sin interés**: todo a principal, ningún gasto.

## D. Ciclos y deuda de tarjeta

18. **compute_cc_cycle cierre simple**: agrupa compras entre cortes, calcula pago total y mínimo.
19. **compute_cc_cycle con pagos parciales**: arrastra saldo y estima interés según `rate_type`.
20. **compra a 1 cuota**: entra al ciclo siguiente sin plan diferido.
21. **compra con installments>1** (si lo habilitas en v1.5): calcula plan y prorratea si aplica.

## E. Reglas

22. **engine prioridad**: respeta `priority` y el primero que calza gana.
23. **scope**: actor sobre household sobre global.
24. **regex y contains**: coinciden con merchant/description y no se aplican falsos positivos.
25. **reglas deshabilitadas**: no se aplican.

## F. Resúmenes y agregaciones

26. **summarize_period**: suma ingresos, gastos, neto, top categorías; excluye transferencias; incluye comisiones como gasto.
27. **multi-currency**: reporta por moneda; bloquea mezcla indebida.

## G. Errores y bordes

28. **falta household**: cualquier caso de uso falla.
29. **currency inválida**: falla.
30. **fecha futura absurda**: política de rechazo o warning.
31. **overflow de amount**: límites razonables.
32. **concurrencia idempotente**: dos hilos con misma clave no duplican.

## H. Contratos de puertos

33. **TransactionRepo contract**: pruebas de contrato con fake repo para garantizar que cualquier implementación cumpla las expectativas.
34. **UnitOfWork atomicidad**: si falla el segundo write, nada queda persistido.
35. **RuleRepo**: orden por prioridad y filtrado por scope.

## I. Propiedad/aleatoriedad ligera

36. **Generador de transacciones** random controlado validando invariantes siempre.
37. **Idempotencia**: N repeticiones con mismas claves nunca cambian el estado final.

## J. Performance del dominio (ligero)

38. **1000 transacciones en memoria**: servicios mantienen latencias sub-milisegundo sin I/O real.

# Orden recomendado de TDD (iteraciones cortas)

1. **Money/Percent/Currency**
2. **Transaction y Category**
3. **Rules engine mínimo**
4. **record_expense** con reglas y idempotencia
5. **Account tipos y Transfer** (+ fee)
6. **Debt + DebtMovement**
7. **record_expense con TC** genera purchase
8. **pay_debt** con split y efectos
9. **compute_cc_cycle** básico
10. **summarize_period** incluyendo exclusión de transfer y suma de fees
11. **Errores y bordes**
12. **Contratos de puertos y UoW**

# Decisiones de diseño y por qué

* **Value Objects para dinero y tasas**: centralizan reglas y evitan bugs de centavos y monedas mezcladas.
* **Debt separada de Account**: tarjetas y préstamos tienen vida y reglas propias; mantenerlas fuera de `Account` evita ifs por todo lado.
* **Transfer entidad propia**: semántica clara para no contar como gasto.
* **Rules engine simple**: suficiente en v1, extensible a ML sin romper el API del core.
* **Ports + UoW**: aislan I/O; el core se testea 95% sin base y sin FastAPI.
* **Idempotencia dentro del core** además del middleware: doble candado, porque los retrys pasan.