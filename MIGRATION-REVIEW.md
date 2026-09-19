# Logseq -> Foam migration review

Converted 66 pages/journals into `/home/mrk/workdir/notes`.

Skipped 49 Logseq built-in schema pages (Query, Task, Property, etc. — framework internals, not notes):

- Alias
- Apply template to tags
- Asset
- Assignee
- Bidirectional property title
- Card
- Cards
- Code
- Comment
- Comments
- Contents
- Deadline
- Description
- Due
- Enable bidirectional properties
- Enable property history
- Extends
- External URL
- Hide empty value
- Hide from Node
- Icon
- Journal
- Library
- Math
- PDF Annotation
- Page
- Page Tags
- Priority
- Property
- Published URL
- Query
- Quote
- Repeating recur frequency
- Repeating recur unit
- Repeating type
- Root Tag
- Scheduled
- State
- Status
- Tag
- Tag Properties
- Tags
- Task
- Template
- Title Format
- User Avatar
- User Email
- User Name
- Whiteboard

Skipped 4 user pages with no body (bare tag markers like `android`/`kafka`, an empty journal day, and one orphaned namespace duplicate):

- Sep 18th, 2026
- android
- cheat sheet⁄mysql⁄admin-queries
- kafka

## Zero-width space cleanup

The old Logseq CLI inserted U+200B (zero-width space) inside `[[ ... ]]` shell test syntax in code blocks, to stop Logseq eagerly parsing it as a wikilink. Foam doesn't have that eager-parse problem, so all U+200B characters were stripped from the converted files — they were invisible but would have broken any script copy-pasted out of a cheat sheet.

## Needs manual attention

None. No `((block-ref))`, `{{query}}`, or `{{embed}}` syntax was found anywhere in the graph.

## Namespace conversion

Logseq's `⁄` (U+2044 fraction slash, used because DB-graphs reject literal `/` in titles) was converted back to a real `/` and turned into actual subdirectories (e.g. `cheat sheet⁄jq` -> `cheat sheet/jq.md`). Wikilinks pointing at those pages were rewritten the same way.

## Tags

Logseq DB-graph "classes" (tags) were kept as a plain `tags:` frontmatter list. Foam has no tag-query system, so tag-based queries/filters from Logseq have no equivalent here.
