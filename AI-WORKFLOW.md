# 🤖 AI-WORKFLOW.md — How I Work With AI

This document explains my philosophy and approach to AI-assisted development. It exists because there's a meaningful difference between using AI and being used by it — and I want to be transparent about which one this is.

---

## My Philosophy

I use AI as a collaborator, not a ghostwriter.

I define the problem. I evaluate the output. I push back when something is wrong. I ask questions when I don't understand. I own every decision.

AI accelerates the journey — it doesn't choose the destination.

When AI suggests something I don't understand, I ask until I do. When it suggests something wrong, I catch it and correct it. When it makes an assumption that doesn't fit my domain, I override it with knowledge it doesn't have.

This is not vibe coding in the negative sense — blindly accepting whatever the model produces. This is guided collaboration where I happen to know billiards, my codebase, and my goals better than the AI ever will.

---

## What AI Helps Me Do Faster

- Scaffold boilerplate I already understand
- Recall syntax I haven't used recently
- Think through tradeoffs by talking them out
- Generate first drafts of documentation
- Identify edge cases I might have missed

## What AI Does Not Do

- Make domain decisions for me
- Choose my architecture
- Define my scope
- Write code I can't explain
- Replace my engineering judgment

---

## Real Examples

### Version Mismatch — league-api Setup
The initial scaffold suggested PostgreSQL packages. I have SQL Server. The suggested NuGet packages defaulted to versions requiring .NET 10. I'm on .NET 8. Two cases where following the AI output blindly would have broken the build. I caught both by reading the output critically rather than accepting it.

### Database Design — DeleteBehavior.Restrict
When building `LeagueDbContext`, AI generated the foreign key relationships. I asked why `DeleteBehavior.Restrict` was needed before accepting it. The explanation — multiple cascade paths from Match to Player causing a SQL Server conflict — made sense once I understood it. I didn't paste code I couldn't defend.

### Scope Decision — Individual vs. Team Leagues
When designing the Match model, the domain complexity of team leagues vs. individual leagues came up. Rather than trying to model both, I made a deliberate scope call: individual handicapped leagues only. AI supported the decision but the reasoning was mine — a portfolio project should showcase clarity, not complexity for its own sake.

### Domain Constraint — Race Length Cap
AI suggested capping race length at 100. I changed it to 20 based on real league knowledge. No league race exceeds 20 games. The AI doesn't know billiards — I do.

### First Working Endpoint — GET /api/players
Built the PlayersController using async/await patterns and mapped directly to PlayerDTO in the query rather than fetching the model and converting after. This keeps the database model from leaking into the API response layer — a deliberate separation of concerns decision, not just a pattern followed blindly.

### EF Core Migration — InitialCreate
Add-Migration generates the schema change as readable C# before touching the database. Update-Database executes it. Reviewed the generated migration file before running it to verify the schema matched the intended domain model. Both Players and Matches tables confirmed in SSMS after migration completed successfully.

### Email Case Normalization — POST /api/player
Email case normalization: AI generated a case-insensitive duplicate check. Developer correctly identified that case-insensitive comparisons are technically wrong at the email protocol level. Solution changed to normalize email to lowercase on input instead — cleaner data storage, exact match comparison. This is an example of domain and protocol knowledge overriding AI output.

### Soft Delete Decision — Player Model
Hard delete rejected because SQL Server's DeleteBehavior.Restrict would refuse deletion of players with match history entirely. Soft delete via IsActive flag chosen because league players go inactive and return — their match history remains relevant across sessions. Domain knowledge drove this decision. IsActive exposed on PlayerDTO to support both active-only views (standings) and full views (admin).

### Migration Default Value Gotcha
Adding a non-nullable column with a default value via EF Core migration sets existing rows to the type default, not the intended application default. IsActive defaulted to false (0) for existing Players rows instead of true (1). Real production scenario would require a data migration script alongside the schema migration to backfill existing records correctly.

### C# Naming Convention Catch — PascalCase Properties
C# properties use PascalCase (IsActive) not camelCase (isActive). Coming from other languages like Dart/Flutter and JavaScript where camelCase is standard, this is an easy slip. Caught during red squiggle review rather than at runtime — a good example of why reading compiler errors carefully matters. JSON serialization will automatically convert PascalCase to camelCase in the API response anyway, so the consumer sees isActive while the C# code uses IsActive.

### Match Validation Rules — Domain Knowledge Over AI
AI would have generated basic existence checks for a POST /api/matches endpoint. The handicap scoring validation — winner's score must equal their race, loser's score must be less than their race — came entirely from domain knowledge. This is a core billiards league rule that makes the difference between an API that accepts data and an API that understands its domain. No amount of prompting would have produced this without the developer knowing the sport.

### Forfeit Handling — CreateMatchDTO
Forfeit scenario identified as a real league use case during DTO design. IsForfeit flag rejected in favor of explicit ForfeitingPlayerId — knowing a match was forfeited isn't enough, you need to know who forfeited. Nullable because most matches complete normally. Forfeit presence triggers alternate validation path: forfeiting player score must be 0, winner score must equal their race, normal handicap validation skipped. Explicit over implicit is the design principle here.

### Forfeit Score Auto-Setting — API Design Decision
Initial implementation required the caller to explicitly set the winning player's score equal to their race on a forfeit. Rejected — the API should enforce and auto-set predictable values rather than requiring the consumer to know internal business rules. If PlayerOne forfeits, PlayerTwo's score is automatically set to their race. Reduces caller complexity and eliminates a category of invalid input entirely.

### Hard Delete vs Soft Delete — Match vs Player
Player deletion uses soft delete (IsActive flag) because league players go inactive and return — their match history remains relevant. Match deletion uses hard delete because an incorrectly recorded match should simply not exist. Same application, two different deletion strategies driven by domain logic, not technical preference. The right deletion strategy depends on what the data represents, not a one-size-fits-all pattern.

### Docker Build vs Local Build — Missing Using Directives
Local Visual Studio builds can succeed without explicit using directives because of implicit usings and IntelliSense filling gaps. Docker builds compile from scratch in a clean Linux container with no IDE assistance — missing using directives that Visual Studio forgave locally become hard build failures in Docker. Always verify your build compiles cleanly in Docker, not just locally.

### Dual Migration Sets — Documented Intent
A production system supporting both SQL Server (local development) and PostgreSQL (containerized deployment) would use separate EF Core migration sets per provider, each in its own folder with its own snapshot file. This was a deliberate architectural decision documented here rather than implemented, because the portfolio value is in understanding the pattern — not the complexity of maintaining two migration histories. The Docker implementation uses PostgreSQL-compatible migrations generated by temporarily forcing the Npgsql provider locally. SQL Server migrations are used for local development via Visual Studio.

### Docker Troubleshooting — Provider Version Compatibility
Npgsql.EntityFrameworkCore.PostgreSQL version must align with EF Core version. Npgsql 8.x is incompatible with EF Core 9.x at runtime despite building successfully. Npgsql 9.0.4 with EF Core 9.0.15 is the correct pairing. Migrations generated against SQL Server use nvarchar — PostgreSQL uses character varying. Provider-specific migrations must be generated with the correct provider active, not just the correct connection string.

### Test Data Strategy — break-and-verify
Test data uses GUID-based unique emails per run to prevent conflicts across multiple test executions. Soft delete via API is called in AfterScenario hooks to keep the active roster clean. Known gap: soft-deleted test players remain visible to admins via GET /api/players?all=true. A production test strategy would address this via a dedicated test data cleanup role, a test-only hard delete endpoint behind an admin auth layer, or a separate test database that gets wiped between runs. Documented as a known limitation rather than implemented — scope decision, not an oversight.

### BDD Test Execution — Lessons From First Run
Six of eight scenarios failed on first run with a single root cause: ReadFromJsonAsync<dynamic> returns JsonElement not a true dynamic object. Property access via dot notation fails silently at runtime. Fixed by deserializing to Dictionary<string, JsonElement> and using GetInt32() for typed access. Second issue: ScenarioContext["Response"] was overwritten by DELETE before list assertion could read it. Fixed by using a separate ScenarioContext["ListResponse"] key for list endpoint responses and a dedicated ThenTheListResponseStatusCodeShouldBe step. Third issue: response stream already consumed before Then step tried to read it. Fixed by reading content to string first then deserializing. All three issues caught and fixed without changing the feature file's business logic — the Gherkin scenarios were correct, the step definitions needed adjustment.

### BDD Step Definition Reusability
Step definitions written for Player Management scenarios were reused directly in Match Management scenarios without modification. SpecFlow resolves step definitions across all classes in the assembly — a step written once is available everywhere. The ThenTheListResponseStatusCodeShouldBe step caught the same ListResponse vs Response context key issue in Matches that was fixed in Players — proof that reusable steps also carry forward lessons learned. Building a solid step library upfront pays dividends as the test suite grows.

### Standings — Streak Display Philosophy
Loss streaks are technically accurate data but not universally preferred in league display contexts. The API correctly computes and returns both win and loss streaks. Display decisions — whether to show loss streaks, reframe them as "games behind", or omit them entirely — belong in the UI layer, not the API. Separation of concerns means the API returns truth and the front end decides how to present it. Documented as a known UX consideration for future portfolio projects that consume this API.

### BDD Test Suite Complete — 32 Scenarios
Three feature files covering the full league-api surface: Players (8), Matches (16), Standings (8). Every API endpoint tested. Every domain validation rule covered in Gherkin. Step definitions built to be reusable across feature files. Test data uses GUID-based emails for isolation, soft delete for cleanup, and ScenarioContext for state sharing between steps.

### Vite as a Build Tool
Vite replaces older tools like Webpack and Create React App. It's faster, simpler to configure, and uses native ES modules in development. The vite.config.ts file is the single source of truth for build configuration — plugins, aliases, server settings, and build output all live there

### UI Color Philosophy — the-practice-log
Blue is the primary color — tournament-grade felt is most commonly blue. Green is secondary — common in recreational pool halls. Gray is accent — emerging in high-profile tournament settings. Color choices are intentional domain decisions, not aesthetic preferences.

### the-practice-log — Documented Intent: Shot Diagram and Cue Ball Position
Two features were scoped out of the portfolio implementation but documented as planned features:
Shot Diagram: A full implementation would use an interactive SVG billiards table where the user places ball positions visually. This requires custom SVG interaction handling, ball placement state management, and a serialization format to store table layouts. Scoped out due to complexity vs portfolio value ratio.
Cue Ball Position (Intended vs Actual): A full implementation would use the same interactive SVG table to capture where the player intended the cue ball to travel vs where it actually ended up. This is the core metric of position play measurement — arguably the most important skill in billiards. Scoped out for the same reason as diagrams. Both features would be the natural next milestone in a production version of this tool.

### the-practice-log — Documented Intent: Cue Ball Contact Point Visual
The cue ball contact point selector uses a dropdown for portfolio implementation. A production version would use an interactive cue ball diagram — a circle divided into 9 zones that the user taps to select where they struck the cue ball. This is significantly more intuitive for billiards players than a dropdown and would be the first UI enhancement in a real release.

### AI Integration — the-practice-log
The AI debrief feature calls the Anthropic Claude API directly from the browser using the claude-sonnet-4-5 model. The prompt includes structured session data — drill type, shot count, made/missed breakdown, and individual shot details including cue ball contact point and power. Initial implementation used markdown formatting which required a custom parser. Simplified by instructing Claude to return plain text with labeled sections instead — less complexity, same readability. The API key is stored in a .env file excluded from version control. The anthropic-dangerous-direct-browser-access header is required for direct browser API calls — in a production system this would be proxied through a backend service to protect the API key.

### API Key Security — Browser vs Server
API keys stored in .env files are bundled into JavaScript by Vite and visible to anyone with browser DevTools. A production implementation would proxy API calls through a backend service — the browser calls your own API, your API calls Anthropic with a server-side key. The league-api would be a natural proxy candidate in a full system architecture.

### Prompt Engineering Over Output Parsing
When AI returns formatted output that's difficult to render, fix the prompt instead of building a parser. Instructing Claude to return plain text with labeled sections is simpler and more reliable than parsing markdown in the browser. This applies to any project where AI output needs to be displayed in a UI — control the format at the source.

### Playwright Strict Mode — Specific Locators
getByText() matches ALL elements containing that text — including dropdown options, headings, and paragraphs. When multiple matches exist Playwright throws a strict mode violation. Fix by using more specific locators: getByRole('heading'), getByRole('button'), getByPlaceholder(), or getByLabel(). Always prefer semantic locators over text matching when elements appear in multiple contexts.

### Playwright vs SpecFlow — Same Concepts, Different Syntax
Playwright (TypeScript) and SpecFlow (C#) follow the same Arrange/Act/Assert pattern. Key differences: Playwright has no Gherkin layer — tests are pure code. Locators replace step definitions — page.getByText(), page.getByRole(), page.getByPlaceholder(). test.beforeEach replaces [BeforeScenario]. The strict mode violation lesson applies universally — always use semantic locators (getByRole, getByLabel) over text matching when elements appear in multiple contexts on the page.

### rack-stats — Data Engineering vs Application Engineering
Data engineering projects have a different architecture than application projects. The pipeline has three distinct layers: data generation (seed.py), computation (queries.py), and presentation (app.py). Each layer is independently runnable and testable. SQLAlchemy plays the same role as Entity Framework — ORM abstracts the database — but Pandas DataFrames replace DTOs as the data transfer mechanism between layers. SQL knowledge transfers directly; the Python syntax is the only new element.

### Synthetic Data Design — Domain Knowledge as a Feature
Generic synthetic data produces generic dashboards. Domain-specific synthetic data produces meaningful analytics. Every design decision in rack-stats — venue names, Fargo ranges, race lengths, entry fees, tournament formats — came from billiards domain knowledge. The resulting dashboard tells a story about Florida billiards culture, not just a data pipeline demonstration. Domain knowledge is a feature, not just a constraint.

### Test Planning — chalk-it-up
Test scenarios defined before writing test code — same discipline as BDD feature files. Three categories: happy path (core user flows), validation (error handling), and locked state (post-match behavior). The locked state tests are particularly important — verifying that completed matches reject further input is a business rule, not just a UI concern. Test planning preceded implementation in every project in this portfolio.

### Widget Test Strict Mode — Flutter vs Playwright
Flutter's widget finder throws an ambiguity error when multiple widgets match — identical to Playwright's strict mode violation. find.text('Allen') matched both the player panel name and the break button label. Fixed with .first to target the specific widget. Same lesson, different framework: always use the most specific finder available. Semantic finders (find.byKey, find.byType) are more robust than text finders when text appears in multiple contexts.

### Universal Testing Lesson — Finder Specificity Across Frameworks
The same strict mode / ambiguity error appeared in three different testing frameworks across this portfolio: SpecFlow (C#), Playwright (TypeScript), and Flutter widget tests (Dart). In every case the fix was identical in concept — use a more specific locator rather than matching on text that appears in multiple contexts. Framework syntax differs; testing principles are universal. When the same lesson appears across multiple projects it becomes a principle, not just a fix.


---

## The Standard I Hold Myself To

If someone asks me to explain any line of code in these projects, I can. If they ask why a decision was made, I have a reason. If the AI got something wrong, I caught it and fixed it — and documented it here.

That's the standard. Everything in this portfolio meets it.