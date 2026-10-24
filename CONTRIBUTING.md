

## Change Scope

Try to keep any given PR within the following limitations

* One completely deployable change with no regressions. 
* "Small". 200 LoC is a rough goal. This excludes:
  * Tests (but test framework changes count)
  * Config
  * Schemas
* The prior goal implies that more that is modeled as config or schema (for example) makes your changes simpler
* Any docs should be considered long-lived.
  * A "here's a project plan" should only ever go under ./.claude/plans. They should never be updated by subsequent changes. Addendum-plans can be added if helpful, or just a completely new plan that doesn't even reference the old.
  * Updates to long lived docs need to take their change to it humbly. If the change is minor to the overall scope, the doc change should also be minor (if any). 
* Planned work counts as "keep it small". Multiple incremental smaller plans is better than one big one. They can be planned as a whole. Each plan should have a simple 1 sentence description which can be read to basically understand the whole plan's goals and approach. 