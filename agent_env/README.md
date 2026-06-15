# ChinaTravel Agent Environment

`agent_env` is a lightweight wrapper that exposes the ChinaTravel benchmark
environment to external agents without changing the official `chinatravel`
package. It provides JSON-friendly access to the existing `WorldEnv`, query
loader, and evaluation flow so an agent can inspect benchmark facts, produce an
itinerary JSON, and save results in the standard `results/<method>/<uid>.json`
format.

The official benchmark code remains the source of truth. `agent_env` is only a
baseline interface layer for tool-calling agents.

## Files

- `adapter.py`: core Python wrapper around `WorldEnv`.
- `cli.py`: command-line interface for one-shot calls or interactive use.
- `mcp_stdio.py`: minimal stdio JSON-RPC/MCP-style tool server.
- `http_server.py`: local HTTP JSON service.
- `scripts/solve_script_with_harness.py`: optional OpenCode/Codex harness that
  loads queries, prompts an agent, saves plans, and evaluates outputs.
- `config.toml.example`: example harness configuration.
- `SKILL.md`: compact instructions for agents that use the CLI.

## Setup

Install the original project requirements and download the ChinaTravel database:

```bash
pip install -r requirements.txt
# unzip the database to chinatravel/environment/database/
```

The wrapper can start without the database, but environment lookups will fail
until the official prerequisites are installed.

## API Surface

### Query Tools

- `china_travel_list_splits`: list local split names from
  `chinatravel/evaluation/default_splits/`.
- `china_travel_load_query`: load query metadata or one query by `split` and
  optional `uid`.
- `china_travel_world_command`: call the original `WorldEnv` command-string API.

### Attraction Tools

- `attractions_keys(city)`: list attraction columns and value types.
- `attractions_select(city, key, op, value)`: filter attractions with `eq`,
  `ne`, `contains`, `lt`, `le`, `gt`, or `ge`.
- `attractions_id_is_open(city, id, time)`: check whether an attraction is open
  at `HH:MM`.
- `attractions_nearby(city, point, topk, dist)`: find nearby attractions.
- `attractions_types(city)`: list attraction types.

### Accommodation Tools

- `accommodations_keys(city)`: list accommodation columns and value types.
- `accommodations_select(city, key, op, value)`: filter accommodations.
- `accommodations_nearby(city, point, topk, dist)`: find nearby accommodations.

### Restaurant Tools

- `restaurants_keys(city)`: list restaurant columns and value types.
- `restaurants_select(city, key, op, value)`: filter restaurants.
- `restaurants_id_is_open(city, id, time)`: check whether a restaurant is open
  at `HH:MM`.
- `restaurants_nearby(city, point, topk, dist)`: find nearby restaurants.
- `restaurants_with_recommended_food(city, food)`: find restaurants by
  recommended dish.
- `restaurants_cuisine(city)`: list cuisines.

### Transport and POI Tools

- `goto(city, start, end, start_time, transport_type)`: query in-city transport.
  `transport_type` must be `walk`, `taxi`, or `metro`.
- `intercity_transport_select(start_city, end_city, intercity_type,
  earliest_leave_time)`: query train or airplane options.
- `poi_lat_lon_search(city, name)`: look up POI coordinates.
- `next_page()`: fetch the next page after a paged `WorldEnv` DataFrame result.

## Usage

List tools:

```bash
python -m agent_env.cli tools
```

Call a tool:

```bash
python -m agent_env.cli call attractions_keys '{"city":"上海"}'
python -m agent_env.cli call restaurants_nearby '{"city":"上海","point":"上海迪士尼度假区","topk":5,"dist":2}'
python -m agent_env.cli call intercity_transport_select '{"start_city":"北京","end_city":"上海","intercity_type":"train","earliest_leave_time":"07:00"}'
```

Use the raw `WorldEnv` interface:

```bash
python -m agent_env.cli world "attractions_keys('上海')"
```

Start an MCP-style stdio server:

```bash
python -m agent_env.mcp_stdio
```

Start the HTTP service:

```bash
python -m agent_env.http_server --host 127.0.0.1 --port 8765
```

HTTP endpoints:

- `GET /health`
- `GET /tools`
- `GET /splits`
- `POST /call` with `{"tool": "...", "arguments": {...}}`
- `POST /world-command` with `{"command": "..."}`

## Harness

`scripts/solve_script_with_harness.py` is the baseline implementation for
running an external agent on a ChinaTravel split. It is intentionally simple:
load a query, remove oracle-only verifier fields from the prompt, ask an agent
harness to solve the task using `agent_env.cli`, parse the final itinerary JSON,
save it in the official result layout, and run the official evaluators.

The baseline currently supports two external agent runners:

- `opencode`: calls `opencode run` with a generated per-run `opencode.json`.
- `codex`: calls `codex exec` with configurable model/provider overrides.

The prompt tells the selected agent to use local CLI calls for exact benchmark
facts such as POI names, prices, opening hours, travel times, TrainID/FlightID
values, and transport costs. The expected answer is a JSON object matching
`chinatravel/evaluation/output_schema.json`, usually wrapped in
`<output>...</output>` tags so the harness can parse it reliably.

Copy and edit the local config:

```bash
cp agent_env/config.toml.example agent_env/config.toml
```

Run the configured harness:

```bash
python agent_env/scripts/solve_script_with_harness.py
```

Common overrides:

```bash
python agent_env/scripts/solve_script_with_harness.py --split easy --limit 1
python agent_env/scripts/solve_script_with_harness.py --split easy --uid <uid>
python agent_env/scripts/solve_script_with_harness.py --harness opencode --model dashscope/qwen3.6-27b
python agent_env/scripts/solve_script_with_harness.py --harness codex --model qwen3.6-35b-a3b
python agent_env/scripts/solve_script_with_harness.py --resume
```

Important config fields:

- `[run].split`: benchmark split to load.
- `[run].limit`: number of split queries to solve, useful for smoke tests.
- `[run].method`: output method name. If empty, the harness uses
  `<model>-<split>-<harness>`.
- `[run].tool_python`: Python command that the agent should use for CLI calls.
- `[run].resume`: skip queries that already have result JSON files.
- `[opencode]` and `[codex]`: model, timeout, API key env var, provider, and
  output parsing settings for each runner.

For each query, the harness writes:

- `prompt.txt`: the exact prompt sent to the agent.
- raw stdout/stderr from OpenCode or Codex.
- `output.txt`: the final agent message.
- `output.json`: the parsed itinerary.
- `results/<method>/<uid>.json`: the official result file.
- `evaluation.json`: one-query schema, commonsense, and logical evaluation.

When running a full split, it also writes
`agent_env/runs/<method>/<split>_summary.json`. Parse failures are recorded as
failed evaluations, so summaries include both invalid JSON outputs and plans
that fail benchmark constraints.
