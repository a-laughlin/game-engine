# src

Paradigm: Functional

Directory Organization:
- Test files are colocated with code: e.g. code.py tests at code.test.py

Architecture Constraints:
- ALWAYS follows functional programming best practices
- ALWAYS follows Onion Architecture. Example: #1 never imports from #2. #2 may import from #1.
  1. utilities-standalone-types: standalone types ONLY for composition by #2
  2. utilities-standalone: standalone utilities ONLY for composition by #3-6
  3. utilities-domain-types: domain-specific utilities ONLY for composition by #4
  4. utilities-domain: domain-specific utilities ONLY for composition by #5-6
  5. pipelines-domain & services-external: ONLY for composition by #6
  6. workflows-domain: ONLY for user interaction


