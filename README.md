# Solution Finder

A Drupal 9/10 module that guides site visitors from the problems they have to the service page that solves them. Visitors tick the pests or issues they're dealing with, and the finder sends them to the best-matching solution page. If nothing matches, it sends them to a fallback destination.

I built it at Coalmarch for **Sprowt**, a multisite Drupal platform for pest control and lawn care companies. It runs on the homepage of many lead-generation sites.

---

## How it works

```
Visitor ticks concerns ──▶ Solution Finder block ──▶ SolutionFinderService::findSolution()
   (e.g. "Termites",           (form + validation)          │
    "Ants")                                                  ▼
                                         Best-matching Solution entity
                                                             │
                                                             ▼
                                Solution page for the visitor's subsite
                                (or the block's default destination)
                                         ?c[]=<concern ids> appended
```

Editors create **Solution** entities. Each solution maps a set of *concerns* (nodes of type `concern` or `issue`) to one or more *solution pages* (package or service pages). When a visitor submits the finder, the module picks the single solution that best covers what they selected. The selected concern IDs go into the query string (`?c[]=…`), so the landing page can personalize its content.

### Matching logic

`SolutionFinderService::findSolution()` resolves a selection with this rule, which runs the same way every time:

1. Find every enabled solution whose concerns include **all** of the selected concerns.
2. From those, keep the solution with the **fewest** concerns attached, since it's the most specific match.
3. If there's a tie, use the **highest priority**, set by drag-and-drop weight on the sort screen.
4. If there's still a tie, use the **most recently updated** solution.
5. If there's still a tie, go by the **alphabetical order** of the labels.

If no solution matches, the visitor goes to the block's **default destination**.

Other modules can change which solutions are considered with `hook_solution_finder_get_solutions_alter(&$solutions, $context)`. For example, they can filter solutions by subsite or by form state.

### Multisite (subsite) destinations

A solution can point to a different page for each Sprowt subsite. Destinations are stored in the `solution_pages` map field, keyed by subsite UUID, with `_main` as the fallback. `Solution::getSolutionPage($sourceNode)` resolves the right page for whichever subsite the finder is placed on.

---

## Features

- **`solution` content entity.** It has a label, status, description, weight, concerns (entity reference) and per-subsite solution pages, and it implements `SolutionInterface`.
- **Solution Finder block.** It's a configurable block plugin that also serves as its own form, built with dependency injection. Its settings are:
  - Heading (formatted text)
  - Maximum number of concerns shown before "More options"
  - Labels for the Submit, More options and Fewer options buttons
  - Concerns to exclude from the checklist
  - Solutions to exclude from matching
  - Default destination (validated as an internal path)
- **Admin UI** at **Structure → Sprowt solutions**:
  - A listing with weights, bulk actions (Views Bulk Operations) and export
  - A drag-and-drop priority screen (DraggableViews) that saves its order back to each entity's `weight`
  - Add and edit forms with a Select2 destination picker grouped by subsite, which shows each page's content type and ID
  - A **Test solution finder** page that runs the matching logic against any combination of concerns and explains which rules decided the result
- **Front-end behaviors.** These are Drupal behaviors built with `once()`. They handle the "More options" toggle, a checked state on each concern tile, and a fully clickable tile area.
- **Integrations:**
  - A Webform options list (`solution_finder_concerns`), so concerns can be reused in other forms
  - Honeypot and time restriction turned off for the finder form, which is a navigation form, not a spam target
  - Translatable button text for Sprowt Translation
  - Block access controlled by the `solution_finder_enabled` Sprowt setting
- **Update hooks** that install the entity type on existing sites and migrate the old single `solution_page` field to the multisite `solution_pages` map.

---

## Requirements

- Drupal 9 or 10
- Core modules: `node`, `block`
- Contrib modules: [DraggableViews](https://www.drupal.org/project/draggableviews), [Views Bulk Operations](https://www.drupal.org/project/views_bulk_operations), `select2_all` (provides the Select2 library used on the solution form)
- Content types: `concern` and/or `issue` for the checklist, plus at least one solution page type: `solution_page_packages`, `solution_page_package_service`, `solution_page_services` or `solution_page_package_alt`
- Optional: Webform, Honeypot, and the Sprowt modules (`sprowt_settings`, `sprowt_subsite`, `sprowt_translation`)

Concern nodes with `field_exclude_from_solution_find` checked are left out of the finder.

## Installation

```bash
# Place the module in your custom modules directory
cp -r pest-solution-finder web/modules/custom/solution_finder

drush en solution_finder -y
drush updb -y   # existing sites: installs the entity type / migrates fields
drush cr
```

Optional configuration in `config/optional/` is imported when its dependencies are present. It includes the Solutions admin view, a `solution_finder` node view mode and the Webform options list.

## Usage

1. Go to **Structure → Sprowt solutions → Add solution**. Choose the concerns the solution covers and the page each subsite should send visitors to.
2. Set priorities on **Structure → Sprowt solutions → Sort** (`/admin/structure/solution/sort`).
3. Place the **Solution Finder** block in a region or in Layout Builder, and set its heading, button text and default destination.
4. Use **Test solution finder** (`/admin/structure/solution/test`) to check which solution a given set of concerns resolves to.

All admin routes require the **Administer solution finder** permission.

## Front-end development

The styles are written in SCSS and compiled to `css/`:

```bash
npm install
npm run build   # compile Sass and run Autoprefixer
npm run watch   # rebuild when SCSS changes
```

## Project structure

```
solution_finder.info.yml          module definition and dependencies
solution_finder.module            hooks: theme, options lists, DraggableViews save, Webform, Honeypot
solution_finder.install           update hooks (entity install, multisite field migration)
solution_finder.routing.yml       admin routes
solution_finder.services.yml      solution_finder.service
src/
  Entity/Solution.php             the solution content entity and its sort rules
  SolutionInterface.php
  SolutionFinderService.php       concern/page lookups and the matching logic
  SolutionListBuilder.php         admin listing with editable weights
  Form/SolutionForm.php           add/edit form with subsite-aware destinations
  Form/SolutionFinderTestForm.php admin tool for testing the matching logic
  Plugin/Block/SolutionFinderBlock.php  the visitor-facing finder (block + form)
js/                               Drupal behaviors (finder UI, Select2 destination picker)
scss/ → css/                      styles
templates/                        Twig templates
config/optional/                  admin view, view mode, Webform options
```

## Author

Matthan Heiselt, Coalmarch
