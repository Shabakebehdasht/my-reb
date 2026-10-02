For h-dashboard PRs: user says 'pr' → create PR from current branch to upstream/beta (asgarimehdi/h-dashboard). All changes commit+push to current branch.
§
Every new session: default cwd is /home/runner/h-dashboard, and always use CodeGraph (`codegraph sync` first; codegraph_explore for code Q&A) + superpowers skills + read-the-damn-docs (web_search official docs) before acting; shadcn/improve for h-dashboard audits only on request.
§
Boost MCP occasionally dies on first stdio call ("lost its stdio subprocess") — just call it again. CLI fallback always works: php scripts/boost_tool.php <tool> '<json>'.
§
h-dashboard branch rebecca tracks origin/beta (branch.rebecca.merge=refs/heads/beta), so `git status` shows 'rebecca...origin/beta'; always use explicit refspecs HEAD:refs/heads/rebecca.
§
MaryUI x-select defaults to optionValue='id'/optionLabel='name'. Options keyed 'value'/'label' need explicit option-value="value" option-label="label" or every <option> renders empty (blank control). Pass :options="$this->myOptions()" from a component method — a bare $myOptions is undefined in the Blade view.
§
scripts/e2e-test.sh: not concurrency-safe, no trap — never run two instances; .env.dev.bak may hold ALREADY-SWAPPED content so .env stays on e2e config (always verify `grep DB_DATABASE .env` == h_dashboard after a run). Lost bak: cp .env.e2e .env, set APP_URL=http://127.0.0.1:8000 + DB_DATABASE=h_dashboard; kill orphan via `kill $(pgrep -f 'artisan serve --port=800[1]')` — never pkill -f 'artisan serve' (kills shared :8000).
§
.env is gitignored; rebuild from `.env-example-github` + secrets in `.env.e2e`, override APP_URL=http://127.0.0.1:8000 and DB_DATABASE=h_dashboard, drop `secrets.` lines, verify `php artisan about --only=environment`. parse_ini_file('.env') fails (unquoted parens) — regex scan or config() instead.
§
Map perf fixed (adc561f): bottleneck was main-thread rendering, not server (longtask /map pan 620→52ms). Fixed: circleMarker+lazy popup, icon cache, id-Map/memo depth, canvas lines, dead Livewire loadStats removed.

