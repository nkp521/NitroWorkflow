# Nitro Workflow Guide

## Table of Contents

- [Creating Tables and Models](#creating-tables-and-models)
  - [Step 1: Create Model First](#step-1-create-model-first)
  - [Step 2: Create Database Migration](#step-2-create-database-migration)
  - [Step 3: Run Migration](#step-3-run-migration)
  - [Step 4: Test Model](#step-4-test-model)
- [Creating GraphQL Queries](#creating-graphql-queries)
  - [Step 1: Create GraphQL Type](#step-1-create-graphql-type)
  - [Step 2: Create Query Resolver](#step-2-create-query-resolver)
  - [Step 3: Register in Schema](#step-3-register-in-schema)
  - [Step 4: Test Query](#step-4-test-query)
- [Creating React Components](#creating-react-components)
  - [Step 1: Create React Component](#step-1-create-react-component)
  - [Step 2: Export Component](#step-2-export-component)
  - [Step 3: Update TypeScript Config (optional?)](#step-3-update-typescript-config)
  - [Step 4: Create Vite Entrypoint](#step-4-create-vite-entrypoint)
  - [Step 5: Load in ERB View](#step-5-load-in-erb-view)
  - [Step 6: Test React Component](#step-6-test-react-component)
- [Common Mistakes](#common-mistakes)
- [Quick Reference](#quick-reference)

---

## Creating Tables and Models

### Step 1: Create Model First

⚠️ **IMPORTANT:** Create the model file BEFORE running migrations. The migration process checks for the model.

**Location:** `components/accounting/app/models/accounting/note.rb`

**Path breakdown:**

- `components/accounting/` - Your component
- `app/models/` - Models folder
- `accounting/` - Namespace folder (critical!)
- `note.rb` - Singular model name

**File path creates namespace:**

- `accounting/note.rb` → `Accounting::Note`
- `accounting/foo/bar.rb` → `Accounting::Foo::Bar`

```ruby
module Accounting
  class Note < Accounting::ApplicationRecord
    belongs_to :project
    belongs_to :user
  end
end
```

**Key conventions:**

- **Module wrapping required:**

  - `module Accounting` creates `Accounting::` namespace
  - Must match component name exactly
  - Prevents conflicts across Nitro

- **Inherit from component's ApplicationRecord:**

  - `Accounting::ApplicationRecord` (not global)
  - Lives at: `components/accounting/app/models/accounting/application_record.rb`
  - Allows component-specific config

- **Model name is singular:**

  - Class: `Note`
  - File: `note.rb`
  - Table: `accounting_notes` (auto-inferred)

- **belongs_to associations:**
  - Defines relationships
  - Validates presence by default
  - Enables: `note.project`, `note.user`
  - Rails finds FK: `belongs_to :project` → looks for `project_id`

---

### Step 2: Create Database Migration

**Location:** `components/accounting/db/migrate/20251001174714_create_accounting_notes.rb`

**Path breakdown:**

- `components/accounting/` - Your component
- `db/migrate/` - Migration folder
- `YYYYMMDDHHMMSS_` - Timestamp (auto-generated)
- `create_accounting_notes.rb` - Descriptive name

```ruby
class CreateAccountingNotes < ActiveRecord::Migration[7.1]
  def change
    create_table :accounting_notes do |t|
      t.integer :project_id, null: false
      t.text :content
      t.integer :user_id, null: false
      t.timestamps
    end

    add_index :accounting_notes, :project_id
    add_index :accounting_notes, :user_id
  end
end
```

**Key conventions:**

- **Table name:** `[component]_[model_plural]` → `accounting_notes`

  - Always plural
  - Component prefix prevents conflicts

- **Class name:** Must match filename in CamelCase

  - File: `create_accounting_notes.rb`
  - Class: `CreateAccountingNotes`

- **Indexes for performance:**
  - `add_index` speeds up lookups
  - Add for any FK or frequently queried column

---

### Step 3: Run Migration

⚠️ **IMPORTANT:** Model file must exist before running migration.

**Command:**

```bash
cd nitro-web  # Main app directory
bin/cobra accounting schema  # Run migrations for accounting component from umbrella
bin/rails db:migrate #Run migration in the umbrella to build the dev database
```

**Important:**

- Must run from **main app directory** (not component directory)
- `bin/cobra [component] schema` is Nitro's migration wrapper
- Targets specific component's migrations
- Creates table in shared Nitro database

**Verify migration ran:**

```bash
# Check migration status
bin/cobra accounting schema:version
```

---

### Step 4: Test Model

**In Rails console:**

```ruby
# Start console
rails c

# Check table exists and columns
Accounting::Note.column_names
# => ["id", "project_id", "content", "user_id", "created_at", "updated_at"]

# Create a test record
note = Accounting::Note.create(
  content: "Test note",
  project: Project.last,
  user: User.last
)

# Verify it saved
note.persisted?  # => true
note.id          # => 1

# Access relationships
note.project.project_number
note.user.full_name

# Query records
Accounting::Note.all
Accounting::Note.where(project_id: 123)
Accounting::Note.order(created_at: :desc).first

# Check validations work
invalid_note = Accounting::Note.new
invalid_note.valid?  # => false
invalid_note.errors.full_messages
# => ["Project must exist", "User must exist"]
```

**Why content shows [FILTERED]:**

```ruby
Accounting::Note.first
# => content: "[FILTERED]"

# This is Nitro's security feature
# Data is saved correctly, just hidden in output
# To see real value:
Accounting::Note.first.content
# => "Test note"
```

---

## Creating GraphQL Queries

### Step 1: Create GraphQL Type

**Location:** `components/accounting/app/graphql/accounting/graphql/accounting_note_type.rb`

**Path breakdown:**

- `components/accounting/app/graphql/` - GraphQL folder
- `accounting/graphql/` - Double namespace folders
- `accounting_note_type.rb` - Format: `[component]_[model]_type.rb` follow the convention in your component

```ruby
module Accounting
  module Graphql
    class AccountingNoteType < NitroGraphql::Types::BaseObject
      graphql_name "AccountingNoteType"
      description "An accounting note type"

      # Database columns
      field :id, ID, null: false
      field :content, String, null: true
      field :project_id, Integer, null: false
      field :user_id, Integer, null: false
      field :created_at, NitroGraphql::Types::DateTime, null: false
      field :updated_at, NitroGraphql::Types::DateTime, null: false

      # Relationships
      belongs_to :user, ::Directory::Graphql::EmployeeType
      belongs_to :project, ::CoreModels::Graphql::ProjectType
    end
  end
end
```

**Key conventions:** FOLLOW YOUR COMPONENT'S CONVENTION

- **Double module wrapping:**

  - `module Accounting` + `module Graphql`
  - Creates: `Accounting::Graphql::AccountingNoteType`
  - Standard pattern in Nitro

- **Class name format:**

  - `[Component][Model]Type`
  - Example: `AccountingNoteType`
  - Includes component name to avoid conflicts

- **Inherit from NitroGraphql::Types::BaseObject:**

  - Nitro's base GraphQL type
  - Shared across all components

- **Field definitions:**

  - Each DB column becomes a field
  - Format: `field :name, Type, null: true/false`
  - Names in snake_case (auto-converts to camelCase in GraphQL)

- **null: false vs null: true:**

  - `null: false` = required (matches `NOT NULL` in DB)
  - `null: true` = optional (allows NULL)
  - Must match migration

- **Relationship types need full namespace:**
  - Start with `::` for root namespace
  - Must find correct component for type
  - Search: `grep -r "class ProjectType" components/*/app/graphql/`

**Type mapping:**
| Database | GraphQL |
|----------|---------|
| `integer`, `bigint` | `Integer` |
| `string`, `text` | `String` |
| `boolean` | `Boolean` |
| `datetime` | `NitroGraphql::Types::DateTime` |
| `date` | `NitroGraphql::Types::Date` |
| `decimal`, `float` | `Float` |
| `json`, `jsonb` | `NitroGraphql::Types::Json` |
| Primary key | `ID` |

**Finding correct relationship namespaces:**

```bash
# Find where ProjectType lives
grep -r "class ProjectType" components/*/app/graphql/
# Result: components/core_models/app/graphql/core_models/graphql/project_type.rb
# Use: ::CoreModels::Graphql::ProjectType

# Find where EmployeeType lives
grep -r "class EmployeeType" components/*/app/graphql/
# Result: components/directory/app/graphql/directory/graphql/employee_type.rb
# Use: ::Directory::Graphql::EmployeeType
```

---

### Step 2: Create Query Resolver

**Location:** `components/accounting/app/graphql/accounting/graphql/accounting_note_query.rb`

**Path breakdown:**

- Same folder as type
- Format: `[component]_[model]_query.rb`

**Basic query (returns all):**

```ruby
module Accounting
  module Graphql
    class AccountingNoteQuery < NitroGraphql::BaseQuery
      description "Returns Accounting Team Notes"

      type [::Accounting::Graphql::AccountingNoteType], null: false

      def resolve
        Accounting::Note.all
      end
    end
  end
end
```

**Query with arguments (filter by project):**

```ruby
module Accounting
  module Graphql
    class AccountingNoteQuery < NitroGraphql::BaseQuery
      description "Returns notes for a specific project"

      type [::Accounting::Graphql::AccountingNoteType], null: false

      argument :project_id, Integer, required: true

      def resolve(project_id:)
        Accounting::Note
          .where(project_id: project_id)
          .order(created_at: :desc)
      end
    end
  end
end
```

**Key conventions:**

- **Double module wrapping (same as type):**

  - `module Accounting` + `module Graphql`

- **Class name format:**

  - `[Component][Model]Query`
  - Example: `AccountingNoteQuery`

- **Inherit from NitroGraphql::BaseQuery:**

  - For queries (read operations)
  - Mutations use `NitroGraphql::BaseMutation`

- **type declaration:**

  - `[Type]` = returns array
  - `Type` = returns single object
  - Full namespace: `::Accounting::Graphql::AccountingNoteType`

- **resolve method:**

  - Contains data-fetching logic
  - Must return data matching `type`
  - Use ActiveRecord queries

- **Arguments:**
  - Define: `argument :project_id, Integer, required: true`
  - GraphQL uses: `projectId` (camelCase)
  - Ruby receives: `project_id:` (snake_case)
  - Auto-converted by Nitro

---

### Step 3: Register in Schema

**Location:** `components/accounting/lib/accounting/graphql.rb`

**Path breakdown:**

- `lib/accounting/` - Library code (not `app/`)
- `graphql.rb` - Standard name

```ruby
module Accounting
  module Graphql
    extend NitroGraphql::Schema::Partial

    queries do
      field :accounting_state_coas, resolver: ::Accounting::Graphql::StateCoasQuery
      field :order_to_cash_bonus_thresholds,
            resolver: ::Accounting::Graphql::OrderToCashBonusThresholdsQuery,
            access: { project_and_installation_services_territory_goals: :view_order_to_cash }
      field :accounting_note, resolver: ::Accounting::Graphql::AccountingNoteQuery
    end

    mutations do
      field :upsert_accounting_state_coa, resolver: ::Accounting::Graphql::UpsertStateCoaMutation
    end
  end
end
```

**Key conventions:**

- **extend NitroGraphql::Schema::Partial:**

  - Makes this a partial schema
  - Nitro merges all components at boot

- **queries block:**

  - All query fields for this component
  - Each `field` becomes available in GraphQL

- **Field naming:**

  - Format: `field :accounting_note`
  - Use snake_case
  - GraphQL converts to: `accountingNote` (camelCase)
  - Pattern: `[component]_[model]` prevents conflicts

- **Resolver specification:**

  - Full namespace: `resolver: ::Accounting::Graphql::AccountingNoteQuery`
  - Leading `::` for root

- **Optional access control:**
  - `access: { resource: :permission }`
  - Integrates with Nitro's permission system
  - If omitted, accessible to all authenticated users

**Why lib/ not app/:**

- Loads at boot time (not runtime)
- Configuration code
- Schema must exist before first request

---

### Step 4: Test Query

**1. Restart server (REQUIRED!):**

```bash
cd nitro-web
# Stop server (Ctrl+C if running)
rails s
```

**Why restart:**

- GraphQL schema loads at boot
- Cached in memory
- Changes need fresh load

**2. Test in GraphQL Playground:**

Navigate to: `http://localhost:3000/graphql/try`

**Basic query:**

```graphql
{
  accountingNote {
    id
    content
    createdAt
    updatedAt
  }
}
```

**With relationships:**

```graphql
{
  accountingNote {
    id
    content
    createdAt
    user {
      id
      firstName
      lastName
      email
    }
    project {
      id
      projectNumber
      ownerName
    }
  }
}
```

**With arguments:**

```graphql
{
  accountingNote(projectId: 3879586) {
    id
    content
    createdAt
    user {
      firstName
      lastName
    }
  }
}
```

**Field name conversion:**

- Schema defines: `:accounting_note` (snake_case)
- Query uses: `accountingNote` (camelCase)
- Same for arguments: `project_id` → `projectId`

**Expected response format:**

```json
{
  "data": {
    "accountingNote": [
      {
        "id": "1",
        "content": "Test note",
        "createdAt": "2025-10-02T15:26:47Z",
        "user": {
          "firstName": "John",
          "lastName": "Doe"
        },
        "project": {
          "projectNumber": "P-123456"
        }
      }
    ]
  }
}
```

---

## Creating React Components

### Step 1: Create React Component

**Location:** `components/accounting/app/javascript/AccountingNoteApp/index.tsx`

**Path breakdown:**

- `components/accounting/` - Your component
- `app/javascript/` - JavaScript/React folder
- `AccountingNoteApp/` - Component name folder (PascalCase)
- `index.tsx` - Main component file

```tsx
import React from "react";

const AccountingNoteApp = () => <div> Capstone Progress </div>;

export default AccountingNoteApp;
```

**Key conventions:**

- **Component folder in PascalCase:**

  - Folder: `AccountingNoteApp/`
  - Component: `AccountingNoteApp`
  - Matches React naming conventions

- **File is always `index.tsx`:**

  - Standard entry point for component folder
  - TypeScript with JSX support (`.tsx` extension)

- **Default export required:**

  - `export default AccountingNoteApp`
  - Allows importing in entrypoint and exporting from index

- **Component naming:**
  - Format: `[Feature]App` or `[Feature]Component`
  - Example: `AccountingNoteApp`, `VolumeRangeApp`, `StateCoa`

---

### Step 2: Export Component

**Location:**
**_Check the package.json for location look for the `main`line _**
`"main": "app/javascript/index.ts",`

`components/accounting/app/javascript/index.ts`

**Path breakdown:**

- `components/accounting/` - Your component
- `app/javascript/` - JavaScript exports folder as per package.json
- `index.ts` - Main export file as per package.json

```typescript
export { default as VolumeRangeApp } from "./VolumeRangeApp";
export { default as GrossMarginApp } from "./GrossMarginApp";
export { default as ImpersonateApp } from "./ImpersonateApp";
export { default as AcknowledgementFormApp } from "./AcknowledgementFormApp";
export { default as AccountingNoteApp } from "./AccountingNoteApp";
```

**Key conventions:**

- **Named exports:**

  - Format: `export { default as [ComponentName] }`
  - Allows importing from package: `@powerhome/accounting`

- **Import path is relative:**

  - `from "./AccountingNoteApp"`
  - Points to component folder (index.tsx is implied)

- **One line per component:**
  - Each React app gets its own export
  - Add new line for each new component

**Why this file exists:**

- Components are imported as: `import { AccountingNoteApp } from "@powerhome/accounting"`
- `@powerhome/accounting` resolves to this file
- Central export point for all component's React apps

---

### Step 3: Update TypeScript Config (optional?)

**Location:** `components/accounting/tsconfig.json`

**Path breakdown:**

- `components/accounting/` - Your component
- `tsconfig.json` - TypeScript configuration

```json
{
  "extends": "../../tsconfig.json",
  // Opt in as components get updated.
  "include": ["./**/StateCoa/**/*.tsx", "./**/AccountingNoteApp/**/*.tsx"]
}
```

**Key conventions:**

- **Extends main config:**

  - `"extends": "../../tsconfig.json"`
  - Inherits Nitro's base TypeScript settings

- **Include specific component folders:**

  - Pattern: `"./**/[ComponentName]/**/*.tsx"`
  - Must add each new React app explicitly
  - Opt-in model (not all components use TypeScript yet)

- **Why explicit includes:**
  - Performance (only compile what's needed)
  - Gradual TypeScript adoption across Nitro
  - Clear which components are TypeScript-enabled

---

### Step 4: Create Vite Entrypoint

**Location:** `app/javascript/entrypoints/accounting/accounting_note.ts`

**Path breakdown:**

- `app/javascript/entrypoints/` - Main app entrypoints folder (not in component)
- `accounting/` - Component-specific subfolder
- `accounting_note.ts` - Entrypoint file (snake_case)

```typescript
import nitroReact from "@powerhome/nitro_react/renderer";
import { AccountingNoteApp } from "@powerhome/accounting";

nitroReact.register({
  AccountingNoteApp,
});
```

**Key conventions:**

- **Import nitroReact renderer:**

  - `from "@powerhome/nitro_react/renderer"`
  - Nitro's React rendering system

- **Import component from package:**

  - `from "@powerhome/accounting"`
  - Uses the export from Step 2
  - Package name matches component folder name

- **Register component:**

  - `nitroReact.register({ AccountingNoteApp })`
  - Makes component available to `render_app` helper
  - Object key must match component name exactly

- **File naming:**
  - Format: `[component]_[feature].ts` (snake_case)
  - Example: `accounting_note.ts`
  - Lives in `entrypoints/[component]/` folder

**Why entrypoints:**

- Vite needs explicit entry points to bundle JavaScript
- Each entrypoint creates a separate bundle
- Loaded only on pages that need them (performance optimization)

---

### Step 5: Load in ERB View

**Location:** `components/accounting/app/views/accounting/project_details/show.html.erb`

**Path breakdown:**

- `components/accounting/` - Your component
- `app/views/` - Rails views folder
- `accounting/project_details/` - Controller namespace
- `show.html.erb` - View file

```erb
<%= vite_client_tag %>
<%= vite_react_refresh_tag %>
<%= vite_javascript_tag "entrypoints/accounting/accounting_note" %>

<%
  @page_title = "#{@project.project_number} Details"
  project_presenter = Projects::ProjectPresenter.new @project, self
%>

<%= stylesheet_link_tag "accounting/project_details/show" %>

<%= pb_rails("button", props: { variant: "link",
                                padding: "none",
                                icon: "long-arrow-left",
                                aria: { label: "Link to Accounting Projects Index" },
                                tag: "a",
                                link: project_details_path }) do %>
  Back to Accounting Projects Index
<% end %>

<%= render(Accounting::ProjectDetails::ShowHeaderComponent.new(project: @project, project_presenter: project_presenter)) %>
<%= render(Accounting::ProjectDetails::ShowProductInstallDetailsComponent.new(project: @project)) %>
<%= render(Accounting::ProjectDetails::ShowFinancingComponent.new(project: @project)) %>
<%= render(Accounting::ProjectDetails::ShowPaymentsComponent.new(project: @project)) %>
<%= render(Accounting::ProjectDetails::ShowProductsComponent.new(project: @project)) %>
<%= render_app "AccountingNoteApp", { projectId: @project.id } %>

<%= javascript_include_tag "accounting/project_details/add_copy_listeners" %>
```

**Key conventions:**

- **Vite tags at top (before Ruby code):**

  1. `vite_client_tag` - Enables Vite dev features
  2. `vite_react_refresh_tag` - Hot reloading for React
  3. `vite_javascript_tag "entrypoints/..."` - Loads your entrypoint

- **Entrypoint path:**

  - Format: `"entrypoints/[component]/[file]"`
  - Example: `"entrypoints/accounting/accounting_note"`
  - No `.ts` extension
  - Path relative to `app/javascript/`

- **render_app helper:**
  - Format: `<%= render_app "[ComponentName]", { props } %>`
  - Component name must match registered name exactly
  - Second argument is props object (optional)
  - Props passed as camelCase: `{ projectId: @project.id }`

**Order matters:**

- Vite tags must be at the very top (before any Ruby code blocks)
- JavaScript must load before React components can render
- `render_app` can be anywhere after Vite tags

---

### Step 6: Test React Component

**Start server:**

```bash
cd nitro-web
rails s
```

**Visit page:**

Navigate to the page with your component. For this example:

```
http://localhost:3000/accounting/project_details/3580709
```

Replace `3580709` with an actual project ID in your database.

**What to check:**

- Component renders on page
- No JavaScript errors in browser console (F12)
- Props are passed correctly (if using React DevTools)
- Hot reload works (edit component, see changes immediately)

**Browser DevTools:**

```bash
# Open browser console (F12)
# Check for errors in Console tab
# Check Network tab for JavaScript bundle loading
# Use React DevTools to inspect component tree
```

**Troubleshooting if not rendering:**

1. Check browser console for errors
2. Verify all Vite tags are present and in correct order
3. Verify component is exported in `app/javascript/index.ts`
4. Verify component is included in `tsconfig.json`
5. Verify entrypoint registers component correctly
6. Restart server (server restart may be needed after new entrypoint)

---

## Common Mistakes

### ❌ Wrong model name

```ruby
Accounting::AccountingNote.all  # Wrong! (includes component name twice)
Accounting::AccountingNotes.all # Wrong! (plural)
Accounting::Note.all            # Correct (singular, component prefix in namespace)
```

````
### ❌ Forgot to restart server

- GraphQL changes require restart
- Schema loads once at boot
- Models/controllers hot-reload, GraphQL doesn't


### ❌ Field name doesn't match column

```ruby
# Database has: t.text :content
field :note_text, String  # Wrong! No column named note_text
field :content, String    # Correct
````

### ❌ File in wrong location

```ruby
# Wrong: components/accounting/app/models/note.rb
# Creates: Note (global namespace - conflict!)

# Correct: components/accounting/app/models/accounting/note.rb
# Creates: Accounting::Note (properly namespaced)
```

### ❌ Missing module wrapper

```ruby
# Wrong - no namespace
class Note < Accounting::ApplicationRecord
end

# Correct - properly namespaced
module Accounting
  class Note < Accounting::ApplicationRecord
  end
end
```

### ❌ Running migration before creating model

```bash
# Wrong order:
# 1. Create migration
# 2. Run bin/cobra accounting schema  ❌ Model doesn't exist yet!
# 3. Create model

# Correct order:
# 1. Create model file
# 2. Create migration
# 3. Run bin/cobra accounting schema  ✅
```

---

## Quick Reference

### File Naming Patterns

| Type             | Location                                        | File Name                           | Class Name                            |
| ---------------- | ----------------------------------------------- | ----------------------------------- | ------------------------------------- |
| Model            | `components/[comp]/app/models/[comp]/`          | `[model].rb`                        | `[Comp]::[Model]`                     |
| Migration        | `components/[comp]/db/migrate/`                 | `YYYYMMDD_create_[comp]_[table].rb` | `Create[Comp][Table]`                 |
| GraphQL Type     | `components/[comp]/app/graphql/[comp]/graphql/` | `[comp]_[model]_type.rb`            | `[Comp]::Graphql::[Comp][Model]Type`  |
| GraphQL Query    | `components/[comp]/app/graphql/[comp]/graphql/` | `[comp]_[model]_query.rb`           | `[Comp]::Graphql::[Comp][Model]Query` |
| Schema           | `components/[comp]/lib/[comp]/`                 | `graphql.rb`                        | `[Comp]::Graphql`                     |
| React Component  | `components/[comp]/app/javascript/`             | `[ComponentName]/index.tsx`         | `[ComponentName]`                     |
| Component Export | `components/[comp]/app/javascript/`             | `index.ts`                          | N/A (exports)                         |
| Vite Entrypoint  | `app/javascript/entrypoints/[comp]/`            | `[comp]_[feature].ts`               | N/A (registration)                    |

### Example (Accounting::Note + AccountingNoteApp)

**Backend:**

- Model: `accounting/note.rb` → `Accounting::Note`
- Migration: `create_accounting_notes.rb` → `CreateAccountingNotes`
- Type: `accounting_note_type.rb` → `Accounting::Graphql::AccountingNoteType`
- Query: `accounting_note_query.rb` → `Accounting::Graphql::AccountingNoteQuery`
- Schema: `accounting/graphql.rb` → `Accounting::Graphql`

**Frontend:**

- Component: `AccountingNoteApp/index.tsx` → `AccountingNoteApp`
- Export: `index.ts` → `export { default as AccountingNoteApp }`
- Entrypoint: `entrypoints/accounting/accounting_note.ts` → Registers component

### Command Reference

```bash
# Run migrations for component (from main app directory)
cd nitro-web
bin/cobra accounting schema #use your component name not accounting

# Check migration status
bin/cobra accounting schema:version #use your component name not accounting

# Rollback last migration
bin/cobra accounting schema:rollback #use your component name not accounting

# Start Rails console
rails c

# Start Rails server
rails s

# Search for GraphQL types
grep -r "class ProjectType" components/*/app/graphql/

# Check what columns exist
rails c
> Accounting::Note.column_names
```

### GraphQL Playground URL

```
http://localhost:3000/graphql/try
```

### Checklist

**Creating Tables and Models:**

- [ ] Create model file FIRST in `app/models/[comp]/[model].rb`
- [ ] Add `module [Component]` wrapper
- [ ] Inherit from `[Component]::ApplicationRecord`
- [ ] Add `belongs_to` relationships
- [ ] Create migration in `components/[comp]/db/migrate/`
- [ ] Use `t.integer` for foreign keys (not `t.references`)
- [ ] Add `add_foreign_key` and `add_index` statements
- [ ] Run `bin/cobra [component] schema` from main app
- [ ] Test in console: create and query records

**Creating GraphQL Queries:**

- [ ] Create type in `app/graphql/[comp]/graphql/[comp]_[model]_type.rb`
- [ ] Use double module: `module [Comp]; module Graphql`
- [ ] Add fields matching database columns
- [ ] Find correct namespaces for relationships (grep search)
- [ ] Create query in `app/graphql/[comp]/graphql/[comp]_[model]_query.rb`
- [ ] Inherit from `NitroGraphql::BaseQuery`
- [ ] Implement `resolve` method
- [ ] Register in `lib/[comp]/graphql.rb`
- [ ] Use snake_case for field name
- [ ] **Restart server**
- [ ] Test in GraphQL playground at `http://localhost:3000/graphql/try`

**Creating React Components:**

- [ ] Create component in `app/javascript/[ComponentName]/index.tsx`
- [ ] Use PascalCase for folder and component name
- [ ] Add `export default [ComponentName]`
- [ ] Export in `app/javascript/index.ts`
- [ ] Update `tsconfig.json` to include component folder
- [ ] Create entrypoint in `app/javascript/entrypoints/[comp]/[feature].ts`
- [ ] Import from `@powerhome/[component]`
- [ ] Register with `nitroReact.register({ [ComponentName] })`
- [ ] Add Vite tags at top of ERB view
- [ ] Add `render_app` call with component name
- [ ] Visit page and verify component renders

---

**End of Nitro Workflow Guide**
