# Planificacion de Tickets — Sistema de Programas Presupuestales

**Creado:** 2026-03-02
**Basado en:** `docs/plans/2026-02-28-sistema-programas-presupuestales-design.md`
**Tickets fuente:** `docs/plans/2026-02-28-tickets-notion.md`

---

## Indice de Sprints

| Sprint | Objetivo | Tickets | Estado |
|--------|----------|---------|--------|
| [Sprint 0](sprint-0/) | Infraestructura y entorno | S0-T1 a S0-T4 | Pendiente |
| [Sprint 1](sprint-1/) | Identidad y aislamiento | S1-T1 a S1-T6 | Pendiente |
| [Sprint 2](sprint-2/) | Cascada de planes y matriz de alineacion | S2-T1 a S2-T11 | Pendiente |
| [Sprint 3](sprint-3/) | Metodologia de Marco Logico (Etapas 1-4) | S3-T1 a S3-T8 | Pendiente |
| [Sprint 4](sprint-4/) | MIR y validaciones | S4-T1 a S4-T10 | Pendiente |
| [Sprint 5](sprint-5/) | Importacion de programas existentes | S5-T1 a S5-T6 | Pendiente |
| [Sprint 6](sprint-6/) | Seguimiento y captura periodica | S6-T1 a S6-T8 | Pendiente |
| [Sprint 7](sprint-7/) | Evaluacion y reportes | S7-T1 a S7-T8 | Pendiente |
| [Sprint 8](sprint-8/) | Orquestacion IA (transversal) | S8-T1 a S8-T3 | Pendiente |

## Dependencias entre Sprints

```
Sprint 0 (Infra)
  └── Sprint 1 (Auth/Teams)
        └── Sprint 2 (Planes/Alineacion)
              ├── Sprint 3 (MML Etapas 1-4)
              │     └── Sprint 4 (MIR/Validaciones)
              │           └── Sprint 5 (Importacion)
              │                 └── Sprint 6 (Seguimiento)
              │                       └── Sprint 7 (Evaluacion)
              └── Sprint 8 (IA - transversal, en paralelo desde Sprint 2)
```

## Registro de Pruebas

El changelog de modificaciones a pruebas se mantiene en:
- [testing/test-changelog.md](testing/test-changelog.md)

## Convenciones

- Cada archivo de ticket sigue la plantilla estandar (encabezado, pasos, archivos, pruebas, criterios)
- Los tests se nombran con el patron: `tests/Unit/` y `tests/Feature/` segun corresponda
- Las modificaciones a pruebas existentes se documentan en el changelog con fecha, ticket y motivo
