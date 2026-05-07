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

---

## The Standard I Hold Myself To

If someone asks me to explain any line of code in these projects, I can. If they ask why a decision was made, I have a reason. If the AI got something wrong, I caught it and fixed it — and documented it here.

That's the standard. Everything in this portfolio meets it.