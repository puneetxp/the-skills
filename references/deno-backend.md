# `the_deno`: Deno runtime (JSR `@puneetxp/the`)

The current release is **`jsr:@puneetxp/the@0.1.16`** (`the_deno/deno.json`). The_billing (INTAX) still pins **`https://deno.land/x/the@0.0.2/mod.ts`**. Check `deno/dep.ts` before you write code, because the two versions differ.

## 1. Bootstrap
```ts
// deno/dep.ts: keep every runtime import here
export { compile_routes, compile_url_pattern, DB, hash, Model, response, Router, Session, setRole } from "jsr:@puneetxp/the@0.1.16";
export type { _Routes, relation, Route_Group_with } from "jsr:@puneetxp/the@0.1.16";

// deno/index.ts
setRole((await Role$().all()).items);                       // load the roles table once, before any session
Deno.serve({ port: 9000 }, async (req) =>
  (await new Router(routes, req).URLPattern()?.run()) ?? new Response("Not Found", { status: 404 }));

// deno/App/Routes/index.ts
export const routes = compile_url_pattern(compile_routes([
  { handler: Public.Home },
  { islogin: true, child: [...islogin, ...isuper] },
  ...Auth,
]));
```
- Run with `deno run --watch --allow-all --unstable-kv index.ts`. KV is required, because sessions live in Deno KV.
- The database connection comes from env or `deno/.env`: `DBHOST`, `DBUSER`, `DBPWD` and `DBNAME`. Version 0.1.x also reads `DBPORT`, `DBPOOL` and `DBSOCKET`.
- **Obsolete APIs (don't use them):**
  - `new Router(routes).route(req)`
  - `/.+` array params
  - `(req, params)` handlers
  - the deno.land/x `0.0.0.4.x` imports

## 2. Routes
Route fields:
- `path`: a URLPattern, e.g. `/:id` or `/book/:book_id/client`.
- `method`: defaults to GET.
- `handler(session, param)`.
- `islogin`, `guard: ((req) => Promise<false | string>)[]`, `roles`, `child`, `group`, and `crud: { class, crud: [letters] }`.

**Guards and roles only run when `islogin` is true** (inherited counts). Guards return `false` to allow the request, or a string to deny it.
```ts
static async show(session: Session, param: URLPatternResult) {
  const id = param.pathname.groups.id;
  const latest = new URL(session.req.url).searchParams.get("latest");
  const body = await session.req.json();             // raw Request = session.req
  return response.JSON((await Client$().find(id)).item, session);
}
```
| Letter | 0.1.x | 0.0.2 |
|---|---|---|
| a / r | GET `/` / GET `/:id` | same |
| c / w | POST `/` / POST `/where` | same |
| u | PATCH or PUT `/:id` | **POST `/:id`** |
| p | PATCH or PUT `/` | PATCH `/` |
| d | DELETE `/:id` + DELETE `/perma_delete/:id` (isuper) | DELETE `/:id` |

Differences in 0.1.x:
- The first matching route wins (in 0.0.2 the last one wins).
- Returns a real 404, and a 500 on exceptions.
- Adds CORS on every response, plus `response.OPTIONS(req)`.
- Supports `Authorization: Bearer` API keys (the `api_keys` table).
- The user with id 1 bypasses role checks.

Generated files:
- `App/Routes/Islogin.ts` exports `islogin`.
- `App/Routes/Isuper.ts` exports `isuper`, with `roles: ["isuper"]`.
- `App/Routes/<Role>.ts` for custom roles.
- Controllers in `App/Controller/<Role>/<Name>Controller.ts`.

**These files are rewritten on every `deno_set()`. Never hand-edit them.** Declare access in the schema (`crud`, with `owner`/`under`/`path`, section 6); put routes that are not CRUD on a table in `App/Routes/index.ts`, which is a template copied once and never rewritten. Add new role route arrays there too.

Controllers are rewritten as well, unless `config.json` has `"table": { "<model>": true }`: then that model's existing controller files are kept (all audiences). Keep the schema letters in step with the methods of a kept class. the_billing keeps account, book, book_asset, book_entry, book_stock, brand, category, client, invoice, journal, journal_detail, product, product_link, server, service, unit and user (list mirrored in its committed `exampleconfig.json`).

```ts
// App/Routes/index.ts (the_billing): non-CRUD routes beside the generated ones
const custom: _Routes = [{
    path: "islogin",
    child: [
        { path: "/business_type", group: { "GET": [{ handler: IsloginBusinessTypeController.all }] } },
        { path: "/book/:book_id/template", child: [
            { path: "/sales", handler: IsloginAccountController.salestemplate },
            { path: "/cabs", handler: IsloginAccountController.cabstemplate },
        ] },
    ],
}];
// route_pre: { islogin: true, child: [...islogin, ...custom, ...isuper, ...manager] }
```

**Check after regenerating:** list every compiled route and diff it against the previous build (and grep the frontend's `/api/...` URLs against it):
```ts
// deno run -A --unstable-kv list.ts file://$PWD/App/Routes/index.ts
const { routes } = await import(Deno.args[0]);
for (const [m, list] of Object.entries(routes)) for (const r of list as any[]) console.log(m, r.pattern?.pathname, r.handler?.name ?? "NO-HANDLER");
```

Generated `delete` handlers soft-delete (`update({deleted_at})`) only when the model has `additional: ["delete"]`, and use `Model.delete` otherwise. compile-php 0.2.24 and earlier always soft-deleted.

## 3. Session
- **Access:**
  - `session.Login`: `{ id, name, email, roles }`
  - `session.ActiveLoginSession`: `{ books, book, session_id, expire, ip, agent, … }`
  - `session.req`
- **Login:** `session.startnew(user, activeRoles, books?, book?)`, then `response.JSONF(session.getLogin().Login, session.returnCookie())`.
- **Keep alive:** `session.reactiveSession()`, or respond with `response.JSONS`.
- **Logout:** `session.removeSession()`.
- **Cookie:** `PHPSESSID`, httpOnly. The domain comes from env `ssl`, plus env `samesite` and `secure`.
- **Bug in 0.0.2:** `SessionRoles` gives every user every role (`.filter` inside `.filter`). 0.1.x fixes it with `.some`. the_billing recomputes the roles in `withRealRoles()` in `AuthController.ts`; keep that workaround while it's on 0.0.2.
- `ActiveLoginSession.book` is always `books[0]`. Scope by the URL's `:book_id` instead.

## 4. Model
```ts
class Standard extends Model<Client> { constructor() { super("client", "clients", nullable, fillable, columns, { book: { table: "books", name: "book_id", key: "id", callback: () => Book$ } }); } }
export const Client$ = () => new Standard();          // current generator: factory (fresh query state per call)
// the_billing (older output): export const Client$ = new Standard();  → shared instance, call without ()
```
```ts
(await Client$().all()).items;  (await Client$().find(5)).item;
(await Client$().where({ book_id: [3] }).andWhereC([["updated_at", ">", latest]]).get()).items;
await Client$().create({...});        // 0.1.x sets .item; 0.0.2 needs .getInserted()
await Client$().where({ id: [5] }).update({...});   // update() without where() updates ALL rows
await Client$().upsert([...]); await Client$().delete({ id: [5] });
await (await Invoice$().where({ id: [1] }).get()).with("client");   // with() is async
```
- Results are on `.item` and `.items`. **There is no `.Item` or `del`.**
- 0.1.x adds `paginate`, `count`, `first`, `softDelete`, `withJoin`, `clone` and `toJSON`.

## 5. Responses
```ts
response.JSON(body, session?, status?, headers?)
response.JSONS(body, session?, status?, headers?)
response.JSONF(body, headers?, status?)
```

## 6. Row scope from the schema: `owner` and `under`
A list entry (`"islogin": ["a","r"]`) generates an **unscoped** controller: `Client$().all()` returns every row. Write the entry as an object to scope it (full rules in SKILL.md 2.A2):

```json
// book.json   "islogin": { "can": ["c","r","u","d","a","w"], "owner": "user_id" }
// client.json "islogin": { "can": ["c","r","u","d","a","p","w"], "under": "book" }
```

Generated route (inside `Routes/Islogin.ts`):
```ts
{ path: "/book", crud: { class: IsloginBookController, crud: [...] },
  child: [{ path: "/:book_id", child: [{ path: "/client", crud: { class: IsloginClientController, crud: [...] } }] }] }
```

Generated controller, every method starts the same way:
```ts
static async all(session: Session, param: URLPatternResult): Promise<Response> {
   const book_id = await ownedParent(session, param, Book$, "book_id", "user_id");   // values from client.json + book.json
   if (book_id instanceof Response) return book_id;                                  // 404: not your book
   const rows = await Client$().where({ book_id: [book_id] }).get();
   return response.JSON(rows.items, session);
}
// show/update/delete: where({ id: [id], book_id: [book_id] }); store/update/upsert: { ...body, book_id }
// where: { ...filters, book_id: [book_id] } so a filter cannot replace the book
```

`App/scope.ts` (template, copied once):
```ts
export async function ownedParent(session, param, parent, key, ownerColumn): Promise<number | Response> {
  const id = Number(param.pathname.groups[key]);
  if (!Number.isInteger(id) || id <= 0) return response.JSON("Not Found", session, 404);
  const row = await parent().where({ id: [id], [ownerColumn]: [session.Login.id] }).first();
  if (!row) return response.JSON("Not Found", session, 404);
  return id;
}
```
- Calls models as factories (`Book$()`, `Client$()`), which is what the current generator writes; `where().first()` exists in 0.0.2 and 0.1.x. Relation callbacks are generated as `() => Book$()` (an instance), which both runtimes accept.
- the_billing uses this since 2026-10-07: client, invoice, account, journal, journal_detail, book_detail, order, book_asset (`path: "asset"`), book_stock (`path: "stock"`) and book_entry (`path: "entry"`) are `under: "book"`; book, server and subscription have `owner: "user_id"`.
- Rename the URL segment with `"path"`: `{ "can": ["c","d","a"], "under": "book", "path": "asset" }` → `/islogin/book/:book_id/asset`.
- `owner` controllers filter `WHERE <owner> = session.Login.id` and set it on create.
- A role uses its own owner: `"roles": { "executive": { "under": "book" } }` checks `book.json` `"roles": { "executive": { "owner": "executive_id" } }`.
- Only generated for `param: "URLPatternResult"`.

### Hand-written per-tenant pattern (inside kept controllers)
```ts
export async function ownedBook(session: Session, param: URLPatternResult): Promise<number | Response> {
  const book_id = Number(param.pathname.groups.book_id);
  if (!book_id) return response.JSON("Book is required", session, 400);
  const book = (await Book$.find(book_id)).item;
  if (!book || book.user_id != session.Login.id) return response.JSON("Not Your Book", session, 403);
  return book_id;
}
// controller: const book_id = await ownedBook(session, param); if (book_id instanceof Response) return book_id;
// then always filter: where({ id: [id], book_id: [book_id] })
```
- the_billing's routes are `/islogin/book/:book_id/{client,invoice,account,journal,journal_detail,asset,stock}`, plus `/islogin/server` (scoped per user) and `/isuper/*`.
- The Deno routes have **no `/api` prefix**. nginx strips it with `proxy_pass http://localhost:9000/;`, and the Vite dev proxy uses `rewrite: p => p.replace(/^\/api/, "")`.

## 7. Checks
Run `deno check index.ts`, or `npx -y deno check index.ts` when Deno isn't installed. the_billing's `deno check App/Routes/index.ts` already has about 23 errors in legacy files.
