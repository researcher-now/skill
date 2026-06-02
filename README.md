# Researcher Skill

Public installable agent skill for Researcher.

Researcher is a hosted, paid research-run service at https://researcher.now.
This repository contains only the public skill contract used by compatible agent
clients. It does not contain the private Researcher product source code and does
not create credentials.

Install:

```bash
npx skills add j-s/researcher-skill --skill researcher
```

Then set a customer API key from https://researcher.now/account/?setup=agent:

```bash
export RESEARCHER_TOKEN="rk_..."
```

The skill lives at `skills/researcher`.

