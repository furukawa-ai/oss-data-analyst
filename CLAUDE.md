# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**oss-data-analyst** is an AI data science agent that converts natural language questions into SQL queries using a multi-phase agentic workflow. Built with the Vercel AI SDK v5, it features streaming responses and a semantic layer for intelligent query planning.

**Note**: This is a reference architecture with simplified sample schemas (Companies, People, Accounts) for demonstration. Production implementations require custom data models.

## Development Commands

### Setup & Initialization
```bash
pnpm install                    # Install dependencies
cp env.local.example .env.local # Set up environment
pnpm initDatabase              # Create SQLite DB with sample data
```

### Development
```bash
pnpm dev                       # Start dev server on port 3000
pnpm dev:tunnel                # Start with ngrok tunnel
pnpm dev:slack                 # Run Slack integration
```

### Build & Type Checking
```bash
pnpm build                     # Build for production
pnpm start                     # Start production server
pnpm type-check                # Run TypeScript compiler checks
pnpm lint                      # Run ESLint
```

### Database Operations
```bash
pnpm db:generate               # Generate Drizzle migrations
pnpm db:migrate                # Run migrations
pnpm db:push                   # Push schema changes
```

### Testing
```bash
pnpm eval:dev                  # Run evalite in watch mode
```
**Note**: No test suite currently exists. Tests would use Vitest (configured in `vitest.config.ts`).

## Architecture

### Multi-Phase Agent Workflow

The agent (`src/lib/agent.ts`) executes queries through 4 sequential phases, each with specialized tools and prompts:

1. **Planning Phase** (`src/lib/tools/planning.ts`, `src/lib/prompts/planning.ts`)
   - Searches semantic catalog for relevant entities
   - Loads entity YAML definitions from `src/semantic/entities/`
   - Assesses data coverage and generates execution plan
   - Tools: `SearchCatalog`, `ReadEntityYamlRaw`, `LoadEntitiesBulk`, `AssessEntityCoverage`, `FinalizePlan`
   - Transitions to Building when `FinalizePlan` tool is called

2. **Building Phase** (`src/lib/tools/building-sqlite.ts`, `src/lib/prompts/building.ts`)
   - Constructs SQL from plan using semantic layer definitions
   - Validates syntax and security policies (no DROP/INSERT/UPDATE/etc)
   - Finds join paths between tables
   - Tools: `BuildSQL`, `ValidateSQL`, `FinalizeBuild`
   - Transitions to Execution when `FinalizeBuild` tool is called

3. **Execution Phase** (`src/lib/tools/execute-sqlite.ts`, `src/lib/prompts/execution.ts`)
   - Estimates query cost
   - Executes SQL with automatic error repair
   - Tools: `EstimateCost`, `ExecuteSQLWithRepair`
   - Transitions to Reporting when `ExecuteSQLWithRepair` tool is called

4. **Reporting Phase** (`src/lib/tools/reporting.ts`, `src/lib/prompts/reporting.ts`)
   - Formats results into tables/charts
   - Performs sanity checks on data
   - Generates natural language explanations
   - Tools: `SanityCheck`, `FormatResults`, `ExplainResults`, `FinalizeReport`
   - Completes when `FinalizeReport` tool is called

### Phase Transitions

Phase transitions are controlled in `src/lib/agent.ts` via the `prepareStep` callback, which:
- Detects when phase-ending tools are called (`FinalizePlan`, `FinalizeBuild`, `ExecuteSQLWithRepair`)
- Updates the `phase` variable
- Returns phase-specific system prompts and active tools

The agent stops when any of these tools complete: `FinalizeReport`, `FinalizeNoData`, `ClarifyIntent`, or after 100 steps.

### Semantic Layer

Located in `src/semantic/`:

- **`catalog.yml`**: High-level entity descriptions for semantic search
- **`entities/*.yml`**: Detailed entity definitions with dimensions, measures, joins
  - Each YAML defines table mappings, field types, descriptions, and relationships
  - Example: `Company.yml` defines the companies table with industry, revenue, employee_count fields
- **`types.ts`**: TypeScript types for semantic layer (`EntityYamlRaw`, `DimensionRaw`, `MeasureRaw`, `JoinRaw`)
- **`io.ts`**: Utilities to load/parse YAML files
- **`schemas.ts`**: Zod schemas for validation

### Database Abstraction

The codebase supports both SQLite (default) and Snowflake:

- **SQLite** (default for demo):
  - Tools: `src/lib/tools/building-sqlite.ts`, `src/lib/tools/execute-sqlite.ts`
  - Client: `src/lib/sqlite.ts`
  - Database file: `data/oss-data-analyst.db`

- **Snowflake** (production):
  - Tools: `src/lib/tools/building.ts`, `src/lib/tools/execute.ts`
  - Client: `src/lib/snowflake.ts`
  - To switch: Update imports in `src/lib/agent.ts` (see commented sections)

### Key Files

- **`src/lib/agent.ts`**: Core agent orchestration with `runAgent()` function
- **`src/app/api/chat/route.ts`**: API endpoint that calls `runAgent()` and streams response
- **`src/lib/security/policy.ts`**: SQL security validation (table allowlists, etc.)
- **`src/lib/sql/validate.ts`**: Syntax validation, blocks DDL/DML operations
- **`src/lib/sql/joins.ts`**: Join path finding between entities
- **`src/lib/execute/repair.ts`**: Automatic SQL error repair logic
- **`src/lib/reporting/viz.ts`**: Visualization generation (charts, tables)

## Adding Custom Tools

1. Create tool file in appropriate phase directory (`tools/planning`, `tools/building`, etc.)
2. Define tool using `tool()` from `ai` package with:
   - `description`: Clear description for the LLM
   - `inputSchema`: Zod schema for parameters
   - `execute`: Async function implementing tool logic
3. Export tool from the phase tools file
4. Import and add to `tools` object in `src/lib/agent.ts`
5. Add tool name to `activeTools` array for appropriate phase in `prepareStep`
6. Update phase system prompt if needed

## Modifying Prompts

System prompts are in `src/lib/prompts/`:
- **`planning.ts`**: Controls entity selection, semantic search behavior
- **`building.ts`**: Controls SQL generation logic and join strategies
- **`execution.ts`**: Controls query execution and error handling
- **`reporting.ts`**: Controls result formatting and visualization choices

Prompts are injected via `prepareStep` in `src/lib/agent.ts`.

## Extending to Production Databases

1. **Switch to Snowflake tools**: In `src/lib/agent.ts`, uncomment Snowflake imports and comment out SQLite imports
2. **Configure credentials**: Set Snowflake connection details in `.env.local`
3. **Update semantic layer**: Replace YAML files in `src/semantic/entities/` with your actual schema definitions
4. **Update catalog**: Modify `src/semantic/catalog.yml` with your entity descriptions
5. **Configure security**: Update `src/lib/security/policy.ts` with your table allowlists

## File Structure

```
src/
├── app/
│   ├── api/chat/route.ts          # Main API endpoint
│   ├── page.tsx                    # Home page with chat UI
│   └── layout.tsx                  # Root layout
├── lib/
│   ├── agent.ts                    # Core agent orchestration
│   ├── prompts/                    # System prompts per phase
│   ├── tools/                      # Agent tools per phase
│   ├── semantic/                   # Semantic layer types and loaders
│   ├── sql/                        # SQL validation, joins, macros
│   ├── security/                   # Security policies
│   ├── execute/                    # Query execution and repair
│   └── reporting/                  # Result formatting and viz
├── semantic/
│   ├── catalog.yml                 # Entity catalog for search
│   └── entities/                   # Entity YAML definitions
├── components/
│   ├── ai-elements/                # Reusable AI UI components
│   ├── chat/                       # Chat interface components
│   └── ui/                         # Shadcn UI components
└── types/                          # TypeScript type definitions

scripts/
├── init-database.ts                # Create SQLite schema
└── seed-database.ts                # Seed sample data
```

## Environment Variables

Required in `.env.local`:
- AI Gateway API key for LLM access (model defaults to `openai/gpt-5`)
- Database credentials if using Snowflake instead of SQLite

## Conventions

- **Commit messages**: Follow conventional commits (`feat:`, `fix:`, `docs:`, `refactor:`, etc.)
- **TypeScript**: Strict mode enabled, use proper types
- **Path aliases**: Use `@/` for imports from `src/` (configured in `tsconfig.json`)
- **Streaming**: Agent uses `streamText()` from AI SDK, responses stream to UI in real-time
