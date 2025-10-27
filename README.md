# Finanzas Core 🏦

Núcleo de dominio para gestión de finanzas personales siguiendo **Domain-Driven Design** y **Arquitectura Hexagonal**.

## ¿Qué resuelve?

- Registrar gastos, ingresos y transferencias
- Modelar deudas (tarjetas de crédito y préstamos) con ciclos y pagos
- Aplicar reglas de categorización automática
- Generar resúmenes por período
- Mantener invariantes: monedas, idempotencia, scoping por hogar

## Arquitectura

```
finanzas_core/
├── entities/          # Entidades de dominio
├── ports/             # Interfaces (repositorios, UoW)
├── services/          # Casos de uso
├── rules/             # Motor de reglas
└── errors.py          # Excepciones
```

## Principios

- ✅ Sin frameworks: código puro Python
- ✅ 100% testeable sin I/O real
- ✅ Ports & Adapters para separar lógica de infraestructura
- ✅ Value Objects inmutables (Money, Currency, Percent)
- ✅ Unit of Work para transaccionalidad

## Conceptos principales

### Value Objects
- **Money**: cantidades monetarias con moneda
- **Currency**: código y decimales (COP, USD)
- **Percent**: valores entre 0 y 1

### Entidades
- **Transaction**: gastos e ingresos
- **Transfer**: movimientos entre cuentas
- **Debt**: deudas y tarjetas de crédito
- **Rule**: categorización automática

## Invariantes del dominio

1. No mezclar monedas en operaciones
2. Todo pertenece a un Household
3. Idempotencia: misma clave = mismo resultado
4. Transferencias ≠ gastos
5. Intereses y comisiones sí son gastos

## Instalación

```bash
poetry install
poetry run pytest
```

## Estado

🚧 En desarrollo - Implementando con TDD según plan de pruebas.

---

**Autor**: Johannes Talero  
**Licencia**: Propietario
