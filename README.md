# Cboe Product Hierarchy

Interactive viewer and editor for the Salesforce dependent picklist hierarchy on the Opportunity object.

**Live URL:** https://jpesenti23.github.io/cboe-product-hierarchy/?v=3
**GitHub Repo:** https://github.com/jpesenti23/cboe-product-hierarchy

## Overview

Extracts the full dependent picklist hierarchy from Salesforce Production and publishes it as a self-contained HTML page on GitHub Pages. Business users can browse the hierarchy and, using Edit Mode, propose changes that export as deployable Salesforce Metadata XML.

## Salesforce Fields (dependency chain)

| Level | Field API Name | Controlling Field |
|-------|---------------|-------------------|
| 0 | `Business_Line__c` | *(controller)* |
| 1 | `Product_Family__c` | `Business_Line__c` |
| 2 | `Product_Subset_1__c` | `Product_Family__c` |
| 3 | `Product_Subset_2__c` | `Product_Subset_1__c` |
| 4 | `Product_Subset_3__c` | `Product_Subset_2__c` |
| 5 | `Product_Subset_4__c` | `Product_Subset_3__c` |

**Org:** jpesenti@cboe.com (Production)

## Features

### Read-Only Viewer
- Tabbed navigation by Business Line (12 total)
- Expandable/collapsible tree with color-coded level badges
- Real-time search with highlight and auto-expand
- Stats cards showing value counts per level

### Edit Mode (added Feb 2026)
- **Add** child values at any level
- **Remove** values (with subtree) after confirmation
- **Rename** values with shared-value detection across parents
- **Move** values between parents via cut-paste
- **Business Line editing** — add, rename, remove entire business lines
- **Undo stack** — up to 50 steps, plus full reset
- **Change summary** — live count of adds/removes/renames/moves
- **Export XML** — generates Salesforce `CustomField` metadata XML for all 6 fields with proper `<valueSettings>` dependency mappings

### Export & Deployment Workflow
1. End user enables Edit Mode, makes changes, clicks Export XML
2. End user downloads the `.field-meta.xml` files and sends to admin
3. Admin places files in `force-app/main/default/objects/Opportunity/fields/`
4. Admin deploys: `sf project deploy start --source-dir force-app`

## Project Files

```
index.html                          # Self-contained HTML page (viewer + editor)
README.md                           # This file
CLAUDE.md                           # Claude Code project instructions
docs/
  Cboe_Product_Hierarchy_Edit_Mode_Instructions.docx  # End-user instructions
```

## How to Update the Hierarchy Data

1. Pull fresh picklist metadata from Salesforce:
   ```bash
   sf sobject describe --sobject Opportunity --target-org jpesenti@cboe.com --json > /tmp/opp_describe.json
   ```
2. Run the Python extraction script to rebuild the `ALL_DATA` JSON and regenerate `index.html`
3. Push to GitHub via API (git push is blocked by corporate proxy):
   ```bash
   # Get current SHA
   SHA=$(gh api repos/jpesenti23/cboe-product-hierarchy/contents/index.html --jq '.sha')

   # Build payload (file too large for command-line args)
   base64 -w 0 index.html > /tmp/index_b64.txt
   python3 -c "
   import json
   with open('/tmp/index_b64.txt') as f: content = f.read().strip()
   json.dump({'message': 'Update hierarchy', 'content': content, 'sha': '$SHA', 'branch': 'main'}, open('/tmp/gh_payload.json', 'w'))
   "

   # Upload
   gh api repos/jpesenti23/cboe-product-hierarchy/contents/index.html \
     --method PUT --input /tmp/gh_payload.json
   ```
4. Bump `?v=N` in the shared URL to bust browser cache

## Technical Notes

- **Single file architecture** — `index.html` is fully self-contained with no external dependencies (no JS libraries, no CSS frameworks, no build step)
- **Data format** — `ALL_DATA` is a nested JSON object: keys are picklist value names, values are child objects (or arrays at leaf level)
- **`validFor` bitmap decoding** — when extracting from Salesforce, empty `validFor` means "not valid for any controller" (not "valid for all"). This was a key bug fix during initial development.
- **Edit mode state** — all edits are in-memory only; refreshing the page reverts to the baked-in `ALL_DATA`. The undo stack stores JSON snapshots (data is ~100KB, so this is fast).
- **XML export** — generates full picklist field definitions, not diffs. Each `<valueSettings>` block lists all controlling field values for a given dependent value. Deploying replaces the entire field metadata.
