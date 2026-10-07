---
name: the-framework
description: >
  Design, build, extend, and debug applications using the "the" framework family by
  puneetxp: the schema-first generator (puneetxp/compile-php + setup.php), the PHP runtime
  (puneetxp/the / the_lib), the Deno runtime (@puneetxp/the / the_deno), the Python/FastAPI
  output, and the Angular (the-angular), SolidJS (the-solid-router), and Vue frontends. Use this skill when working in
  repositories prefixed with `the_`, `the-`, or `the`, or in apps built on them (e.g. the_billing / INTAX).
  Trigger on mentions of: 'the framework', 'the_lib', 'compile-php', 'setup.php', 'autosetup.php',
  'TPHP', 'puneetxp/the', 'the_deno', '@puneetxp/the', 'the-angular', 'the-solid-router',
  `database/Model/*.json`, `config.json` "back-end"/"front-end", or role namespaces isuper/islogin/ipublic.
---

# The Framework ("the")

A schema-first framework family created by Puneet Sharma (`puneetxp`). You describe each table **once** in JSON (`database/Model/*.json`). The generator `puneetxp/compile-php` then writes the SQL DDL, a role-guarded REST backend (Python FastAPI, Deno, PHP, .NET, Go, or Java Spring), and typed frontend code (SolidJS, Angular, or Vue). The generated code runs on small per-language runtimes that share the same concepts: a `Model` ORM with eager relations, a router with `islogin`/`roles`/`guard`, and a CRUD shorthand.

---

## 0. Repository Map (under `/Users/waseemakram/Downloads/puneetxp/` & `vendor/puneetxp`)

| Repo / Folder | Role | Package / Status |
|---|---|---|
| `compile-php` | **The generator**, plus the HTML→PHP view compiler | Composer `puneetxp/compile-php` (PSR-4: `Puneetxp\CompilePhp\`). Active. |
| `the_lib` | PHP runtime (`The\` namespace) | Composer `puneetxp/the` (tag 0.1.304, PHP ≥ 8.2). Active. |
| `the_deno` | Deno runtime | JSR `@puneetxp/the` 0.1.16 (HEAD). Apps may still pin deno.land/x `the@0.0.2`. |
| `the_template_php` | Starter template (`composer create-project`) | Composer `puneetxp/the_template_php`. |
| `the-angular` | Angular UI library | npm `the-angular` 0.0.13, Angular 21. Active. |
| `the-angular-material` | Angular 16 predecessor | Legacy. |
| `the_web_component` | Vanilla custom elements (`<editable-list>`, `<wysiwyg-bs>`) | Prototype. |
| `the_dotnet` / `the_dotnet_api` | .NET 10 runtime port and sample API | **Not generated.** `dotnetset.php` exists but is never called (section 4). |
| `the_go` | Go/Gin helper skeletons | **Not generated.** `golangset.php` exists but is never called (section 4). |
| `the_spring` | Java Spring Boot helper skeletons | **Not generated.** `javaspringset.php` exists but is never called (section 4). |
| `the_billing` | INTAX GST billing app (Deno + SolidJS + MySQL) | The reference Deno app at `/Users/waseemakram/Documents/the_billing`. |
| `intaxing23` | intaxing.in production app (PHP + Angular + MySQL) | The reference PHP app. |
| `apac-genaiacademy-c2` | Rural farming platform (Python/FastAPI + SolidJS + PostgreSQL), at `/Users/waseemakram/Documents/apac-genaiacademy-c2` | The gold-standard Python/FastAPI reference app, and source of `python/app/core`. |
| `the_doc` | Hugo/Doks documentation site | `content/en/docs/**` |
| `skills` | **This skill.** Clone of `puneetxp/the-skills`; `~/.claude/skills/the-framework` symlinks here, and intaxing23 tracks it as a submodule. | Active. Edit here, then commit and push. |
| `compiley-php` | Older snapshot of the generator, no git | Ignore it; use `compile-php`. |
| `the` | 2022 start pack | Historical. |
| `thesolidmarket` | Solid demo app | Historical. |
| `prompts`, `references` | Loose notes next to the clones, not in any repo | Scratch. `skills/references/` is the tracked copy. |

---

## 1. Generator Workflow & CLI

1. **Declare schema in `database/Model/<name>.json`**. **Never hand-edit generated SQL or models.**
2. Run `php setup.php` at the project root. It executes `Puneetxp\CompilePhp\setup`:
   ```php
   use Puneetxp\CompilePhp\setup;
   require "./vendor/autoload.php";

   $setup = new setup(__DIR__);
   $setup->config();          // table_set + back-end generators + front-end generators + write()
   
   // Specialized operations:
   // $setup->migrate();      // Direct DB migration (drops DB first if "fresh": true)
   // $setup->sync();         // Introspects DB, auto-adds missing columns & FK constraints!
   // $setup->migratealter(); // Runs alter queries from database/mysql/alter/*.sql
   ```
3. **Database execution**:
   - `database/Migration.sql` contains combined table DDL, foreign keys, and role seed inserts.
   - For PostgreSQL: `psql -U $DBUSER -d $DBNAME -f database/Migration.sql`.
   - For MySQL: files consolidated in `database/mysql/{structure,relations,insert,alter}/`.
4. **PHP View Compilation**:
   - `php compile.php` executes `Puneetxp\CompilePhp\Compile\Compile`, parsing `<t-...>` HTML templates and `@props({...})` into PHP class components extending `\The\PageBase`.

---

## 2. Model JSON Schema Specification

```json
{
  "name": "client",                     // Singular model name (used for classes & types)
  "table": "clients",                   // Plural database table name
  "crud": {
    "isuper":  ["c","r","u","d","a","p","w"],                         // list = letters only, all rows
    "islogin": { "can": ["c","r","u","a","w"], "under": "book" },     // object = letters + which rows (see 2.A2)
    "ipublic": ["r","a"],
    "roles":   { "executive": { "can": ["r","a"], "under": "book" } }
  },
  "enable": 1,                          // Adds enable TINYINT(1) DEFAULT 1; generates toggle() in frontend
  "additional": ["slug","seo","delete"],// Presets: slug (VARCHAR 255), seo (title + seo_description), delete (deleted_at)
  "default": ["id", "created_at", "updated_at"], // Table lifecycle columns override
  "unique": ["email", ["user_id", "client_code"]], // Single & composite unique keys
  "data": [
    { "name": "name",   "mysql_data": "varchar(255)", "datatype": "string" },
    { "name": "email",  "mysql_data": "varchar(255)", "datatype": "string", "default": "NULL" },
    { "name": "status", "mysql_data": "varchar(20)",  "datatype": "string", "sql_attribute": "NOT NULL DEFAULT 'draft'" },
    { "name": "price",  "mysql_data": "decimal(10,2)","datatype": "number", "default": "0.00" },
    { "name": "notes",  "mysql_data": "text",         "datatype": "string", "default": "NULL" },
    { "name": "tags",   "mysql_data": "text[]",       "datatype": "array",  "default": "NULL" },
    { "name": "meta",   "mysql_data": "jsonb",        "datatype": "json",   "default": "NULL" }
  ],
  "relations": {
    "user": { "name": "user_id", "key": "id", "table": "users" }
  }
}
```

### A. CRUD Operations
- `c`: **Create** (`POST /`, status 201)
- `r`: **Read single** (`GET /{id}`)
- `u`: **Update** (`PUT /{id}` or `PATCH /{id}`)
- `d`: **Delete** (`DELETE /{id}`)
- `a`: **Read all** (`GET /`, supports `?latest=<updated_at>` delta sync)
- `w`: **Where filter** (`POST /where` with `{ filters }`)
- `p`: **Upsert / Batch** (`POST /upsert` or `PUT /`)

### A2. Row scope: `owner` and `under` (Deno + SolidJS generators)
A list says **what** an audience can do. It does not say **which rows**: a list-only `islogin` controller is the same as `isuper` and returns every row. To scope rows, write the entry as an object:

| Entry | Meaning | Deno route | Generated filter |
|---|---|---|---|
| `["r","a"]` | all rows (unchanged) | `/<audience>/<model>` | none |
| `{ "can": [...], "owner": "user_id" }` | row is mine when `<owner>` = my login id | `/<audience>/<model>` | `WHERE user_id = Login.id`; `create` sets it |
| `{ "can": [...], "under": "book" }` | row is mine when its parent is mine | `/<audience>/book/:book_id/<model>` | `ownedParent()` check, then `WHERE book_id = :book_id`; `create`/`update`/`upsert` set it |
| `{ "can": [...], "under": "book", "path": "asset" }` | same, served at another URL segment | `/<audience>/book/:book_id/asset` | as above (`path` works with `owner` and plain objects too) |

Rules (no defaults, nothing guessed):
- `under: "book"` means model `Book$`, URL param and column `book_id` (the same `<name>_id` rule as relations).
- The parent's owner column is read from the **parent's entry for the same audience**: `client.json` `"islogin": { "under": "book" }` uses `book.json` `"islogin": { "owner": "user_id" }`; `"roles": { "executive": { "under": "book" } }` uses `book.json` `"roles": { "executive": { "owner": "executive_id" } }`. One table can have a different owner per audience.
- Generation **stops** if the parent has no `owner` for that audience, if the parent schema does not exist, or if `public`/`ipublic` uses `owner`/`under`.
- `owner` is one column, so one person per row per audience. Many people per row (a manager assigned to many clients) needs a link table; not supported yet.
- Public access is the key `"public"`. The Deno generator does not read `"ipublic"`; an `ipublic` entry generates nothing for Deno.

```json
// book.json
"crud": {
  "isuper":  ["c","r","u","d","a","p","w"],
  "islogin": { "can": ["c","r","u","d","a","w"], "owner": "user_id" },
  "roles":   { "executive": { "can": ["r","a"], "owner": "executive_id" } }
}
// client.json
"crud": {
  "isuper":  ["c","r","u","d","a","p","w"],
  "islogin": { "can": ["c","r","u","d","a","p","w"], "under": "book" },
  "roles":   { "executive": { "can": ["r","a","w"], "under": "book" } }
}
```

### A3. Hand-written code next to generated code (Deno)
`php setup.php` rewrites every generated file, every run. Three rules keep custom work safe (compile-php 4d25b63+):

| What | Where it lives | Generator |
|---|---|---|
| Controller with real logic (invoice numbering, journal balancing, validation) | its normal file `App/Controller/<Role>/<Name>Controller.ts` | skipped when `config.json` `"table": { "<name>": true }`; applies to every audience of that model |
| Route that is not CRUD on a table (`/book/:book_id/template/sales`, `/business_type`) | `App/Routes/index.ts`, in a `custom` route array next to `...islogin` | never written (template copied once) |
| Access to a table (which audience, which letters, which rows, which URL) | the model JSON `crud` block | writes `App/Routes/Islogin.ts`, `Isuper.ts`, `Ipublic.ts`, `<Role>.ts` |

- **Never hand-edit the generated route files**; the next run erases the edit. Declare it in the schema, or put it in `Routes/index.ts`.
- A kept controller must have a method for every letter the schema grants (`a`→`all`, `r`→`show`, `c`→`store`, `u`→`update`, `d`→`delete`, `w`→`where`, `p`→`upsert`), or the route has no handler. Match the letters to the class.
- A kept file can re-export another class under the generated name: `export { IsloginBookAssetController as IsloginBook_assetController } from "./BookHoldingController.ts";`
- `config.json` is usually gitignored (DB credentials), so mirror the keep-list in a committed `exampleconfig.json`, or a fresh clone regenerates over the custom controllers.
- **Models are factories**: `export const Client$ = () => new Standard()`, and relation callbacks are `() => Book$()`. Hand-written code calls `Client$().where(...)`, never `Client$.where(...)`. Each call gets its own query state, which fixes the shared-builder bug (filters from one request leaking into another) on the_deno 0.0.2.
- Generated `all()` passes `param` (`Client$().all(param)`), which the_deno 0.1.x uses; on 0.0.2 it is a type-only mismatch (`TS2554`), ignored at runtime.

How it is wired in compile-php: `index.php` keeps each entry as written in `$table['access']` and reduces `$table['crud']` to letters, so every other generator (PHP, Python, Angular, Vue, .NET, Go, Spring) reads `crud` exactly as before and ignores `owner`/`under`. `denoset.php` reads `access`; `solidset.php` reads `access.islogin.under`. Python keeps its own rules in `app/core/ownership.py`.

### B. Relations (3 Valid Formats)
1. **Simple String Array (Default 1:N convention)**:
   `"relations": ["user", "farm"]`
   Automatically creates `user_id BIGINT UNSIGNED NOT NULL` and `farm_id BIGINT UNSIGNED NOT NULL`, appends foreign keys, and configures ORM relationships.
2. **Object Array with Alias and Nullability**:
   `"relation": [{"name": "category", "alias": "service_category_id", "default": "NULL"}]`
   or multiple relations to the same table:
   `"relation": [{"name": "product", "alias": "primary_product"}, {"name": "product", "alias": "secondary_product"}]`
3. **Explicit Keyed Map (Recommended for enterprise / predefined `data` schemas)**:
   ```json
   "relations": {
     "crop": { "name": "crop_id", "key": "id", "table": "crops" },
     "farmer": { "name": "farmer_id", "key": "id", "table": "users" }
   }
   ```
   If the foreign key column (e.g. `crop_id`) is already declared in `"data"`, `explicitRelation()` binds directly to it without creating duplicate columns.

### C. Default Rules (`default` at 3 Levels)
1. **Table-Level Lifecycle (`defaultsetup`)**:
   Controls automatic system columns for the table. Defaults to `["id", "created_at", "updated_at"]`.
   - `id`: `BIGINT UNSIGNED PRIMARY KEY AUTO_INCREMENT` (MySQL) or `BIGSERIAL PRIMARY KEY` (PostgreSQL), `fillable: false`.
   - `created_at`: `TIMESTAMP DEFAULT CURRENT_TIMESTAMP NOT NULL`, `fillable: false`.
   - `updated_at`: `TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP NOT NULL`, `fillable: false`.
   - Overrides:
     - `["id", "created_at"]`: Immutable event logs or append-only ledgers.
     - `["updated_at", "id"]`: Active session / state tables (e.g. `active_role.json`).
     - `["id"]`: Lean pivot / bridge tables without timestamp columns.
2. **Column-Level Default & Nullability**:
   - `"default": "NULL"` or `"default": null`: Generates `DEFAULT NULL`, TypeScript `T | null`, and Python `T | None = None`.
   - `"default": 0`, `"default": 1`, `"default": "'draft'"`: Sets SQL default value.
   - Any column without `"default": "NULL"` and without explicit `NULL` in `sql_attribute` is forced to `NOT NULL`.
3. **Relation-Level Foreign Key Nullability**:
   - `"relation": [{ "name": "client", "default": "NULL" }]`: Marks the auto-generated foreign key column as nullable (`DEFAULT NULL`).

### D. Built-in Schema Presets (`additional` & `enable`)
- `"enable": 1` or `"enable": true`: Injects `enable TINYINT(1) DEFAULT 1` (`datatype: 'number'`).
- `"additional": ["slug"]`: Injects `slug VARCHAR(255) NOT NULL`.
- `"additional": ["seo"]`: Injects `title VARCHAR(255) DEFAULT NULL` and `seo_description LONGTEXT DEFAULT NULL`.
- `"additional": ["delete"]`: Injects `deleted_at TIMESTAMP DEFAULT NULL`. Triggers soft-delete behavior in Deno and ORM services (`update({ deleted_at: new Date() })`).

### E. Model Types (`type`)
- Standard database table (default when omitted).
- `"type": { "name": "photo", "version": { "thumb": { "width": 200, "quality": 80 }, "large": { "width": 800, "quality": 90 } } }`:
  In PHP backend, generates image upload handler using `FileAct` and automated multi-size WebP resizing via `Img::webpImage()`.
- `"type": { "name": "file" }`: Generates binary file upload handling.

### F. Unique Constraints (`unique`)
- Single column: `"unique": ["email"]`
- Composite columns: `"unique": [["user_id", "crop_id"]]`
- Generates `UNIQUE KEY <name> (<cols>)` in MySQL and `CONSTRAINT <name> UNIQUE ("<cols>")` in PostgreSQL.

---

## 3. Database Engines & Schema Synchronization

### PostgreSQL Support (`"postgresql": true` in `config.json`)
- Native data type conversions:
  - `varchar(N)` → `VARCHAR(N)`
  - `decimal(M,D)` → `DECIMAL(M,D)`
  - `vector(N)` → `vector(N)` (PostgreSQL `pgvector` support)
  - `bigint` / `bigint unsigned` → `BIGINT`
  - `int` / `integer` → `INTEGER`
  - `tinyint(1)` → `SMALLINT`
  - `timestamp` → `TIMESTAMP`
  - `date` → `DATE`
  - `text` / `longtext` → `TEXT`
  - `text[]` → `TEXT[]` (PostgreSQL array type)
  - `json` / `jsonb` → `JSONB`
  - `boolean` → `BOOLEAN`
- Attribute stripping: Removes MySQL-specific keywords (`UNSIGNED`, `AUTO_INCREMENT`, `COMMENT`). Converts `id` with auto-increment to `BIGSERIAL PRIMARY KEY`.
- Output: Emits unified `database/structure.sql`, `database/relation.sql`, `database/insert.sql`, and `database/Migration.sql`.

### MySQL Support (Default)
- Emits individual table files in `database/mysql/{structure,relations,insert,alter}/` plus root consolidated files.

### Schema Synchronization (`$setup->sync()`)
- Inspects live database via `information_schema.columns` and `information_schema.KEY_COLUMN_USAGE`.
- Automatically executes `ALTER TABLE ADD COLUMN` for any newly added schema columns.
- Automatically executes `ALTER TABLE ADD CONSTRAINT FOREIGN KEY` for missing foreign key constraints.

### Schema Deltas (`database/Model/Additional/*.json`)
- Additional table extensions placed in `database/Model/Additional/*.json` are read by `$setup->add_table()`.
- Automatically generates `database/mysql/alter/<name>_alter.sql` with `ADD COLUMN IF NOT EXISTS` and merges attributes into memory.

---

## 4. Backend Generators

### A. Python / FastAPI (`pythonset.php`)
- Target directory: `python/app/`
- Architecture:
  - `app/models/<snake>.py`: Pydantic model (`BaseModel`) and request input schema (`<Model>Input`) with all fields optional and server-managed fields excluded.
  - `app/orm/<snake>.py`: ActiveRecord `Model` subclass with explicit `fillable` array and lazy-loading `relations` map using lambda imports (`lambda: __import__('app.orm.<module>', fromlist=['<Class>']).<Class>`).
  - `app/services/<snake>_service.py`: Service class extending `CrudService` with singleton getter.
  - `app/api/<scope>/<model>/<model>.py`: FastAPI `APIRouter` controllers:
    - `isuper`: Guarded by `dependencies=[Depends(get_current_admin)]`.
    - `islogin` & custom roles: Enforces `owner=current_user` scoping (`current_user=Depends(get_current_active_user)`).
  - `app/api/routers.py`: Auto-aggregates `all_routers` list for clean mounting in `app/main.py`.
  - Implicit namespace packages (PEP 420, no `__init__.py` clutter).

### B. Deno (`denoset.php`)
- Target directory: `deno/App/`
- Architecture:
  - `App/Model/<Name>.ts`: Subclasses `Model<T>` from `@puneetxp/the`.
  - `App/controller/<Role>/<Name>Controller.ts`: Static CRUD methods (`all`, `where`, `show`, `store`, `update`, `upsert`, `delete`).
  - Soft delete: Sets `deleted_at: new Date()` when model specifies `additional: ["delete"]`; exposes `perma_delete` for `isuper`.
  - Routing: `App/routes/<Role>.ts` exporting route arrays with roles. Supports `param: "URLPatternResult"` or `param: "string[]"`.
  - Row scope (2.A2): `owner` / `under` entries get a scoped controller (URLPatternResult only); `under` children are nested at `/<parent>/:<parent>_id/` inside the parent's route (`path` renames the segment). The shared check `ownedParent()` lives in `App/scope.ts` (template, copied once).
  - Kept controllers, factory models, and where custom routes go: 2.A3.

### C. PHP (`phpset.php`)
- Target directory: `php/App/`
- Architecture:
  - `App/Model/<Name>.php`: Subclasses `The\Model` with `$nullable`, `$fillable`, `$relations`.
  - `App/Controller/<Role>/<Role><Name>Controller.php`: REST controller methods.
  - Photo / file processing: Built-in `Img::webpImage` and `FileAct` for automatic multi-size thumbnailing.
  - Routes: `php/Routes/pre/api/<Role>.php` compiled to `Routes/web.php` via `php php/set.php`.

> **D, E and F below are unreachable dead code as of 0.2.25.** The classes exist in
> `src/Class/`, but nothing ever instantiates them: `setup.php` exposes no
> `dotnet_set()` / `golang_set()` / `spring_set()`, and `config()` branches only on
> `deno`, `php`, `python`, `angular`, `solidjs` and `vuets`. Running `php setup.php`
> produces **no** .NET, Go or Spring files. The descriptions record what the classes
> would emit if they were wired up — do not promise these outputs to anyone.

### D. .NET / C# (`dotnetset.php`) — not wired up
- Intended target directory: `dotnet/`
- Would generate C# class models in `dotnet/Models/<Name>.cs` and ASP.NET Core `[ApiController]` controllers in `dotnet/Controllers/<Name>Controller.cs`.

### E. Go / Gin (`golangset.php`) — not wired up
- Intended target directory: `go/`
- Would generate Go structs with `json:"..."` tags in `go/models/<name>.go` and Gin controller handlers in `go/controllers/<name>_controller.go`.

### F. Java / Spring Boot (`javaspringset.php`) — not wired up
- Intended target directory: `spring/`
- Would generate JPA entities (`@Entity`, `@Table`, Lombok `@Data`) in `spring/.../model/<Name>.java`, Spring Data JPA repositories in `repository/<Name>Repository.java`, and `@RestController` in `controller/<Name>Controller.java`.

---

## 5. Frontend Generators

### A. SolidJS (`solidset.php`)
- Target directory: `solidjs/src/shared/`
- Architecture:
  - `Interface/Model/<Name>.ts`: TypeScript interfaces with nullability mapping (`json` → `any`, `array` → `string[]`, `vector` → `number[]`).
  - `Service/Services.ts`: Instantiates typed `ModelService<T>`:
    ```typescript
    export const CropService = (new ModelService<Crop>())
      .seTable("crop")
      .seturl("/islogin/crop/");
    ```
    For `"islogin": { "under": "book" }` it also emits a URL helper, and the app sets the book before use:
    ```typescript
    export const clientUrl = (book_id: number | string) => `/islogin/book/${book_id}/client/`;
    export const ClientService = (new ModelService<Client>()).seTable("client").seturl("/islogin/book/:book_id/client/");
    // app: ClientService.seturl(clientUrl(activeBookId()))
    ```
    The IndexedDB cache is per table, not per book: filter by `book_id` in the UI when the user switches books.
    Preserves hand-written re-exports across regenerations (`export * from "./Weather"`).
  - `run.ts`: Central initializer executing `Service.checkinit()`, caching into IndexedDB on `isLogin()` and purging with `indexdb.The_clearData()` on logout.

### B. Angular (`angularset.php`)
- Target directory: `angular/src/app/shared/`
- Architecture:
  - `Service/Model/<Name>.service.ts`: Angular `@Injectable` service with RxJS observables (`name$`), delta-sync via `?latest=`, and toggle method for `enable`.
  - `Ngxs/State/<Name>.state.ts` & `Ngxs/Action/<Name>.action.ts`: Full NGXS State and Action declarations (`Set`, `Add`, `Edit`, `Delete`, `Upsert`).
  - `Form/Validation/<Name>.ts`: Reactive Form validation definitions (`Create<Name>Form`, `Update<Name>Form`) with `Validators.required` accurately computed based on column nullability.
  - `Service/run.service.ts`: Initialization service with timeouts.
  - Patches `angular.json` with custom assets and output paths.

### C. Vue (`vueset.php`, called via `vuejs_set()`)
- Enabled by `"vuets"` in `config.json`'s `front-end` list. Unlike D/E/F this one *is* reachable.
- Target directory: `vuets/src/shared/`
- Architecture:
  - `Store/Model/<Name>.js`: Pinia store (`defineStore`) with getters, mutation actions (`addItem`, `removeItem`, `editItem`, `upsertItem`), and HMR support (`import.meta.hot`).
  - `Service/Model/<Name>.js`: REST service invoking Pinia store methods.

---

## 6. HTML Component Compiler (`Puneetxp\CompilePhp\Compile`)

- Source files: `src/Compile/Compile.php`, `htmlParser.php`, `phpCompile.php`.
- Compiles custom HTML components into PHP class components extending `\The\PageBase`.
- Syntax:
  - Components: `<t-component-name>`, `<t-php.if>`, `<t-php.for>`.
  - Props declaration: `@props({"condition": false, "child": false})`.
  - Templating: `{{ $expression }}` compiled to `<?= $expression ?>`.
  - Directives: `@foreach($items as $item) ... @endforeach`.

---

## 7. Configuration (`config.json`)

```json
{
  "fresh": false,
  "postgresql": true,
  "back-end": ["python"],
  "front-end": ["solidjs"],
  "env": {
    "dbhost": "localhost",
    "dbuser": "postgres",
    "dbpwd": "password",
    "dbname": "cropsense_db",
    "port": "5432"
  },
  "table": {
    "crop": false,
    "user": false
  }
}
```

- **`postgresql: true`**: Switches generator to PostgreSQL dialect (`BIGSERIAL`, `JSONB`, `vector`, `TEXT[]`).
- **`fresh: true`**: Drops and recreates database on migration.
- **`table` map**: `"crop": true` = hand-written: the Deno generator never overwrites that model's existing controllers (all audiences). `false` or a missing key = generated, rewritten every run. `setup->write()` adds `false` for every new table. See 2.A3.

---

## 8. Development Protocol & Best Practices

1. **Schema is the Single Source of Truth**: Always edit `database/Model/<name>.json`. Never manually modify generated models, DDL, or routers.
2. **Execute Full Pipeline**:
   ```bash
   ./pipeline.sh generate        # Runs setup.php
   ./pipeline.sh test-backend    # Verifies Python ORM & core tests
   ./pipeline.sh build-frontend   # Verifies TypeScript types & Vite build
   ./pipeline.sh check           # Full 5-point verification across all stacks
   ```
3. **Additive Schema Modifications**: Never drop or destructively rename existing columns in production tables. Add new fields with `"default": "NULL"` or sensible defaults.
4. **Ownership & Security**:
   - Every user-facing table must say who owns its rows. **Deno:** write the `islogin` (and each role) entry as an object with `owner` or `under` (2.A2); a plain list is NOT scoped and returns every row. **Python:** `app/core/ownership.py` (default `user_id`, `OWNERSHIP`, `PARENTS`, `SHARED_READ`).
   - Python `/islogin/` endpoints enforce ownership automatically; Deno ones only when the schema says `owner`/`under`. Admin-only endpoints belong strictly in `isuper`.


---

## 9. Known issues (check before relying on a feature)

Verified against compile-php `0.2.25-5-g4d25b63`. Per-platform detail lives in the
reference files; this table is the short list to check before promising a feature.

| Where | Issue | Status |
|---|---|---|
| `the_lib` `Auth::profile()` / `profileupdate()` | Filter on `user_id`, a column `users` does not have. `Model::where()` silently drops unknown keys, so GET returns the **first user** and POST updates **every user**. | **Open, security.** Fix is `where(["id" => [$_SESSION['user_id']]])`, allowing only name/phone/email. Ask before editing `the_lib`. See `references/php-backend.md`. |
| `the_lib` `Auth::login` | A wrong password falls through to 404 "User Not Found" — the `Response::why(...)` result isn't returned. | Open. Keep the 404 if you fix it; the-angular treats any 2xx as success. |
| `Model::where()` (PHP + Deno) | Unknown column keys are dropped, and `update()` with no where updates every row. | By design. Always use real column names and array values: `["id" => [$id]]`. |
| PHP template <= 0.2.24 | `"ilogin" => true` typo left `/api/env` and `/api/isuper` **unprotected**; `/reset` routed to a missing method. | Fixed in later templates (login/register moved to `Inotlogin.php`). Existing projects need a hand fix. |
| `dotnetset` / `golangset` / `javaspringset` | Classes exist in `src/Class/` but are never instantiated — no `setup.php` method, not in `config()`. **They emit nothing.** | Open. See the note in section 4. |
| `php_set()` | Rewrites `php/env.php` from `config.json` on every run, discarding hand edits. intaxing23's `env.php` has `samesite "None"` while its config says `"Strict"`. | Keep `config.json` in sync before regenerating. |
| Deno `the@0.0.2` | `SessionRoles` grants every user every role; update verb is POST; no 404. | Fixed in 0.1.x. the_billing works around it with `withRealRoles()`. |
| the_billing | Hand-edited `Routes/Islogin.ts` and `Isuper.ts` are overwritten by `deno_set()`. Its Deno routes have no `/api` prefix (nginx strips it). | Save those files before regenerating. |
| Python ownership | A table with no `OWNERSHIP` rule and no `user_id` is unscoped for `/islogin/*` — any signed-in user reads and writes every row. | By design (shared data). Add a rule, or grant `islogin` only `r`/`a`. |
