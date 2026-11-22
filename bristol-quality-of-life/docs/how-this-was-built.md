# Bristol Quality of Life: Page 6 build notes

Project: Bristol Quality of Life (Power BI PBIP project)
Page: "Bristol Quality of Life" (Page 6 of the report)
Status: working first version; refinements and a GitHub portfolio write-up to follow.

## 1. What the page does

A survey-driven choropleth of Bristol's 34 wards. The viewer picks a survey question from a flyout panel, picks a year, and the page updates:

- The **page title** becomes the selected question, with the year shown beneath it.
- The **ward map** (Deneb, Vega-Lite) shades each ward from a dark navy to a light blue ramp, with ward labels and a bottom-left legend. It uses the same heat-map style as Page 5 (supermarkets).
- A **ranked table** lists the wards from highest to lowest result for that question and year.
- The **tooltip** shows the ward, its result, the Bristol average and the difference ("vs Bristol"), or "No data".

## 2. Data

Source: Bristol Quality of Life survey, 2024/25 release (CSV, about 22 MB, long format: one row per ward, indicator and year).

Trim (done in Power Query so the model only holds what the report can use):
- Kept only rows that have a ward code and a ward value (`Ward_e_codes` and `Ward_Group_Statistic` not blank).
- Removed 14 unused columns (sample sizes, standard errors, confidence limits, p-value, ICS fields, FID and similar).
- Result: 42,364 rows and 11 columns, covering 34 wards, 293 questions, years 2018 to 2024.

Remaining columns: Year, Indicator_Reference_Number, Public_Title, Theme, Polarity, Bristol_average, Ward_e_codes, Ward_Group_Statistic, Sample_Size_response, Statistical_Significance, Gap.

Model:
- Relationship: survey `Ward_e_codes` to `Sheet[Ward ID]` (many-to-one, single direction). The join is on the ONS ward code (E05...), not the ward name, because the survey's ward names differ from the names in `Sheet`.
- Five measures in the `Key Measures` table (display folder "Survey"): Survey Value, Survey Title, Survey Year, Survey Bristol Average, Survey Significance.
- Design rule: the map only shows a value when exactly one question is selected. The question count and the year are evaluated at question level using `ALL('Sheet')` so a ward selection can't change them. The year is the one chosen in the year selector.

## 3. Page layout (canvas 1280 x 720, dark background #10171e)

| Element | Position and size | Notes |
|---|---|---|
| "Choose question" button | x16, 180 x 40, top row | opens the flyout |
| Title card | x212, width 876 | shows the selected question |
| Year subtitle card | under the title | shows the year, for example 2024 |
| Year selector | right of the top row, ending at x1264 | dropdown, single select |
| Ward map (Deneb) | y92 to y708 | left and centre of the page |
| Ranked ward table | x932, y92, 332 x 616 | right-hand column |
| Question flyout panel | exactly over the table (x932, y92, 332 x 616) | hidden on first load |

Flyout panel group (all inside one group, hidden by default): background and header, the question slicer (hierarchy Theme > Public_Title, single select), and a close button with a cross.

Design choice: the flyout opens over the table's area so the map is never covered while a question is being chosen.

## 4. How the flyout works

- Two bookmarks: "QoL - Show question panel" and "QoL - Hide question panel".
- Each bookmark captures **Display only**: Data is off, Current page is off, and Selected visuals is on (the group and its three children). This stops a bookmark from resetting the selected question or year.
- The open button and the close button both have a Bookmark action pointing at the matching bookmark.
- In Desktop's edit view, a bookmark button needs Ctrl+click. In reading view a plain click works.
- Power BI can't close a flyout automatically when a slicer item is clicked, which is why the close button is needed.

## 5. How Claude Code was used

Two Claude tools split the work by layer:

| Layer | Tool | What it did |
|---|---|---|
| Semantic model (tables, relationship, measures, trim) | Claude Desktop with the Power BI Modeling MCP server, working on the live open model | Edited the Power Query, added the relationship, wrote and tested the measures |
| Report layer (page, visuals, bookmarks, Deneb spec) | Claude Code in the VS Code terminal, run in the project folder, with the Power BI report authoring skill (powerbi-report-cli) and the pbir command line tool | Created the page, visuals, flyout group, bookmarks and the Deneb map, and validated the result |

Working method:
1. **Brief in a file.** The instructions for Claude Code were written to `claude_prompt_page6.md` in the project folder (not pasted into the terminal), so the brief is versioned with the project and can be re-read. Fixes used a second brief, `claude_prompt_page6_fix.md`.
2. **Restore points.** Before each round, a backup folder was made (`_backup_before_page6_2026-11-21`, and later folders for the bookmark and panel fixes), so any round can be undone.
3. **Desktop closed.** Claude Code edits the PBIR files on disk, so Power BI Desktop was closed while it worked and reopened afterwards to check the result visually.
4. **One layer each.** Claude Code was told not to edit the semantic model. The model was only changed through the live MCP session.
5. **Validation.** Claude Code ran `pbir validate --all` after each round. The three errors it reports were already in the project (a Page 3 field-type mismatch and two Innovate theme registration entries in `report.json`). It also listed every file it changed, and compared against the backup to prove nothing else moved.
6. **Direct JSON edits.** pbir cannot write some things, so three were edited directly: the map's internal viewport size, each bookmark's panel visibility and "selected visuals only" setting, and the map folder name.

Lessons learned (useful for the portfolio write-up):
- **Visuals inside a group use positions relative to the group**, not page coordinates. The first build used page coordinates and the panel rendered about 80px too low. Fixed by giving the group page bounds and the children relative positions.
- **Bookmarks that show or hide a group need explicit visibility state.** The first version left each visual's state empty, so Desktop had nothing to apply. The fix writes the group's hidden or visible state, with the panel children hidden too.
- **Keep the brief consistent.** One brief had a layout line and a context line that contradicted the later instruction, which cost a round. Re-read a brief for contradictions before running it.
- **Check what Desktop shows, not just what validation says.** Validation passed while the panel was misplaced, so the visual check in Desktop is a required step.
- **Plugin install:** `pip install pbir-cli` kept dropping mid-download, so the file was downloaded with curl, checked against PyPI's checksum, and installed. It is not on PATH; it lives in the Python Scripts folder.

## 6. Known issues and choices to revisit

- The ranked table headers still read "Name" and "Survey Value" (could be "Ward" and "Result").
- The flyout is 332px wide, so long question titles wrap or truncate.
- Title card is 50px tall; very long questions may need more height.
- The Year selector is single-select (chosen so the map always shows one year).
- Page 5's report-page tooltip was removed from the copied map; this page uses its own Deneb tooltip.
- Possible extra analysis: a deprivation-decile breakdown. The rows removed by the trim can be restored by editing the Power Query filter.
- Pre-existing validation errors listed in section 5.

## 7. Next steps

1. Refine the page (list the issues above, then fix in a Claude Code round using a brief file).
2. Give Page 5 the same dark background.
3. Fix the two broken sources so a full refresh works.
4. Tidy for GitHub: add a README, a screenshot of the page, and this note; exclude the backup folders; check the data licence before publishing the CSV.
5. Explore a Fabric app version (see the chat for the trade-offs).
