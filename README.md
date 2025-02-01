# Paperbell

## Current Features

- Monitors file creation and extracts `institute` information from yaml frontmatter to create notes
- Provides global API for `Quickadd` integration
- Displays or automatically creates notes after retrieving information
- Supports custom templates for institution notes

## Template Variables

You can use the following template variables in your institution note templates:

| Variable | Description | Example |
|----------|-------------|---------|
| {{paperbell.name}} | Institution's full name | Harvard University |
| {{paperbell.abbr}} | Institution's abbreviation | HU |
| {{paperbell.aliases}} | List of alternative names | - Harvard<br>- Harvard College |
| {{paperbell.website}} | Institution's website URL | https://harvard.edu |
| {{paperbell.location}} | Geographic coordinates | [42.3744, -71.1169] |
| {{paperbell.logo}} | Institution's logo URL | https://example.com/logo.png |
| {{paperbell.tags}} | List of tags | - university |

### Using Templates

1. Create a template file in your vault (e.g., `templates/institution.md`)
2. Use the variables above in your template file
3. Set the template path in plugin settings
4. The plugin will use your template when creating new institution notes

Example template:
````markdown
---
name: {{paperbell.name}}
abbr: {{paperbell.abbr}}
aliases: 
{{paperbell.aliases}}
website: {{paperbell.website}}
location: {{paperbell.location}}
tags: 
{{paperbell.tags}}
---

# {{paperbell.name}}

## Overview

[Website]({{paperbell.website}})

## Affiliated Scholars

```dataviewjs
let name = dv.current().name

dv.table(["Name", "Title", "Website", "Email"],
dv.pages(`#scholar`)
  .where(b => b.institute.includes(name))
  .map(b => [b.file.link, b.title, ("[🔗]("+b.website+")"), b.email])
  .sort(b => b.paper_date, 'desc')
)
```
````
