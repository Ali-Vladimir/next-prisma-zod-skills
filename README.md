# next-prisma-zod-skills

Skills para agentes de IA (Claude Code, Codex, Cursor, Copilot y cualquier agente compatible con
`SKILL.md`) enfocadas en construir sistemas web con **Next.js (App Router) + Prisma + Zod +
Server Actions**.

Las skills están escritas en **inglés**, porque es el idioma en el que los modelos siguen
instrucciones con más precisión. La documentación del repo, las explicaciones y la comunidad
están en **español**.

## Skills

| Skill | Qué hace |
| --- | --- |
| [`nextjs-prisma-crud`](nextjs-prisma-crud/SKILL.md) | Construye, modifica o revisa módulos CRUD: modelo Prisma, schema Zod, Server Actions, página, manager, formulario y tabla. Hace que el agente respete las convenciones de tu repo y aplique siempre autenticación, permisos, scope por tenant y transacciones en el servidor. |

### Qué obliga a hacer al agente

1. **Descubrir** tu proyecto antes de escribir código: gestor de paquetes, versiones,
   layout, helper de auth, campo de tenant (`companyId`, `accountId`...), auditoría y forma de
   borrar.
2. **Leer un módulo análogo** completo y copiar sus patrones en lugar de inventar una
   arquitectura nueva.
3. **Construir** capa por capa siguiendo las reglas del servidor: validar con Zod, autenticar,
   autorizar, filtrar por tenant cada consulta e ID relacionado, y agrupar escrituras
   relacionadas en una transacción.
4. **Verificar y reportar**: typecheck, lint y tests, diciendo qué pasó, qué falló y qué no se
   ejecutó. Nunca corre migraciones, deploys ni cambios de dependencias sin que se lo pidas.

Los detalles (paginación, transacciones, Zod 3 vs 4, Prisma 7, integraciones como email,
WhatsApp, Cloudinary y cron) están en [`references/`](nextjs-prisma-crud/references) y el
agente los carga solo cuando la tarea los necesita.

## Instalación

Con el CLI [`skills`](https://github.com/vercel-labs/skills):

```bash
npx skills add Ali-Vladimir/next-prisma-zod-skills
```

Manual (Claude Code): copia la carpeta `nextjs-prisma-crud` en `.claude/skills/` de tu proyecto,
o en `~/.claude/skills/` para tenerla en todos tus proyectos.

## Uso

No hace falta invocarla: el agente la activa solo cuando le pides algo como:

- "Agrega el CRUD de proveedores con su página de administración."
- "Agrega el campo `descuento` a ventas y actualiza el formulario."
- "Revisa las actions de pagos, quiero saber si un usuario puede tocar datos de otra empresa."

También puedes llamarla por nombre: `/nextjs-prisma-crud`.

## Skills recomendadas de Prisma

Para la parte de Prisma (CLI, Client API, setup de base de datos, migración a Prisma 7) usa las
skills oficiales, que se mantienen al día con cada versión:

```bash
npx skills add prisma/skills
```

`nextjs-prisma-crud` se encarga de cómo se arma la aplicación; las de Prisma, de la referencia
del ORM. Se complementan.

## Contribuir

Ideas, errores y mejoras son bienvenidas. Lee [CONTRIBUTING.md](CONTRIBUTING.md) antes de abrir
un PR: explica cómo se escriben las skills de este repo.

## Créditos

- La forma de escribir las skills sigue los principios de
  [`writing-for-agents`](https://github.com/mattpocock/skills/tree/main/skills/productivity/writing-for-agents)
  de Matt Pocock.
- Skills oficiales de Prisma: [prisma/skills](https://github.com/prisma/skills).

## Licencia

[MIT](LICENSE)
