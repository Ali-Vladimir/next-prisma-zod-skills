# Cómo contribuir

Gracias por querer mejorar estas skills. Este documento explica cómo están escritas para que
los cambios mantengan el mismo estilo.

## Idioma

- **Skills (`SKILL.md` y `references/`)**: en inglés.
- **README, issues, PRs y discusiones**: en español.

## Estructura de una skill

```text
nombre-de-la-skill/
  SKILL.md          ← lo que el agente lee siempre que la skill se activa
  references/       ← detalle que se carga solo cuando hace falta
```

`SKILL.md` lleva frontmatter con `name` y `description`. La `description` está cargada en el
contexto del agente **todo el tiempo**, así que cada palabra cuesta: di qué hace la skill y
cuándo usarla, con un disparador por caso distinto, sin sinónimos.

## Principios de escritura

Basados en [`writing-for-agents`](https://github.com/mattpocock/skills/tree/main/skills/productivity/writing-for-agents)
de Matt Pocock:

1. **Pasos con criterio de terminado.** Cada paso termina en una condición verificable
   ("Done when you can state X, Y, Z"), no en algo vago como "entender el proyecto".
2. **Divulgación progresiva.** En `SKILL.md` va solo lo que todos los casos necesitan. Lo que
   solo algunos casos usan va a `references/`, con un enlace que diga *cuándo* leerlo.
3. **Una sola fuente de verdad.** Cada regla vive en un solo lugar. Si una regla aparece en
   `SKILL.md`, los references la enlazan en vez de repetirla.
4. **Instrucciones en positivo.** Escribe lo que el agente debe hacer. Las prohibiciones se
   reservan para límites duros (migraciones, deploys, producción) y van acompañadas de la
   alternativa.
5. **Nada que el modelo ya haga solo.** Si una frase no cambia el comportamiento del agente
   ("escribe código limpio", "usa Git"), se borra.
6. **No copies lo que el repo ya dice.** El agente puede leer `package.json` o la estructura de
   carpetas; la skill guarda lo que no se ve: convenciones implícitas, razones y trampas.
7. **El repo del usuario manda.** Las skills llenan huecos; no imponen una arquitectura nueva.

## Antes de abrir un PR

- Prueba la skill en un proyecto real con tu agente y cuenta en el PR qué tarea le diste y qué
  cambió en su comportamiento.
- Si agregas un ejemplo de código, verifica que corre con las versiones actuales de Next.js,
  Prisma y Zod (o indica la versión).
