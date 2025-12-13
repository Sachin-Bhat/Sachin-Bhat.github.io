+++
title = "Devlog"
description = "Running notes on what I’m building and learning."
+++

# Devlog

Short, informal updates about what I’m tinkering with—across tech, finance, pricing/risk toys, agentic tooling, and systems experiments. No fixed cadence: I post when there’s something interesting to share.

## Entries
{% for page in section.pages | sort(attribute="date") | reverse %}
- [{{ page.title }}]({{ page.permalink | safe }}) — {{ page.date | date(format="%Y-%m-%d") }} — {{ page.description | default(value="") }}
{% endfor %}
