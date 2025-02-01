# Paperbell

## Current Features

- Automatically monitors file updates to extract `institute` information from yaml frontmatter
- Provides global API for `Quickadd` integration
- Displays or automatically creates notes after retrieving information
- Supports custom templates for institution notes

## How to use

1. Install the plugin
2. Create a template file in your vault (e.g., `templates/institution.md`)
3. Use the variables above in your template file
4. Set the template path in plugin settings
5. The plugin will use your template when creating new institution notes

Or use default command/ribbon icon to search institution and create note.

## Template Variables

You can use the following template variables in your institution note templates:

| Variable | Description | Example |
|----------|-------------|---------|
| {{ppb.institute.name}} | Institution's full name | Harvard University |
| {{ppb.institute.abbr}} | Institution's abbreviation | HU |
| {{ppb.institute.aliases}} | List of alternative names | - Harvard<br>- Harvard College |
| {{ppb.institute.website}} | Institution's website URL | https://harvard.edu |
| {{ppb.institute.lat}}, {{ppb.institute.lon}} | Geographic coordinates | 42.3744, -71.1169 |
| {{ppb.institute.logo}} | Institution's logo URL | https://example.com/logo.png |
| {{ppb.institute.tags}} | List of tags | - university |

### Using Templates

1. Create a template file in your vault (e.g., `templates/institution.md`)
2. Use the variables above in your template file
3. Set the template path in plugin settings
4. The plugin will use your template when creating new institution notes

Example template:
````markdown
---
abbr: {{ppb.institute.abbr}}
aliases:
{{ppb.institute.aliases}}
website: {{ppb.institute.website}}
location: [{{ppb.institute.lat}}, {{ppb.institute.lon}}]
logo: {{ppb.institute.logo}}
name: {{ppb.institute.name}}
tags:
{{ppb.institute.tags}}
---

# {{ppb.institute.name}}

## Overview

[Website]({{ppb.institute.website}})

```mapview
{"name":"Default","mapZoom":8,"centerLat":{{ppb.institute.lat}},"centerLng":{{ppb.institute.lon}},"query":"","chosenMapSource":0}
```

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
