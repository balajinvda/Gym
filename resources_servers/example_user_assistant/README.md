# User–Assistant Simulation

This example runs a multi-turn conversation between two independently
configured Responses API agents:

- `example_assistant` is the policy being evaluated.
- `example_user` simulates a customer with a private goal.
- `example_user_assistant_processor` owns turn ordering, termination, and
  trajectory attribution.
- `example_user_assistant` owns task-scoped shared state and verification.

The user wants a vegetarian meal costing at most $20. The user agent persists
that requirement through `save_preference`; the assistant later reads the
shared state through `read_preferences` and saves a recommendation. The user
accepts the recommendation through `accept_recommendation`, which terminates
the episode. The verifier rewards only an accepted recommendation that matches
the persisted diet and budget.

## Why orchestration is a processor

Neither participant owns `/run`. Each agent only handles `/v1/responses` and
may use its own instructions, tools, model, and tool-call loop. The
`UserAssistantProcessor` owns the complete episode:

```text
rollout collector
  → UserAssistantProcessor /run
      → resources server /seed_session
      → assistant agent /v1/responses
      → resources server /episode_status
      → user agent /v1/responses
      → resources server /episode_status
      → ... until environment termination or max_turns
      → resources server /verify
```

Cookies are forwarded through every call, so tools used by either participant
operate on the same task-scoped resources-server state.

## Run the example

Configure `policy_base_url`, `policy_api_key`, and `policy_model_name` in
`env.yaml`, then run:

```bash
.venv/bin/gym eval run \
  --config resources_servers/example_user_assistant/configs/example_user_assistant.yaml \
  --agent example_user_assistant_processor \
  --split validation \
  --output results/example_user_assistant.jsonl
```

The rollout row contains:

- `response`: assistant-only Responses API output for evaluation and training.
- `assistant_trajectory`: exact request/response pairs for assistant turns.
- `user_trajectory`: exact request/response pairs for simulated-user turns.
- `episode_trajectory`: ordered participant outputs, state snapshots, and the
  explicit termination event.
- `termination_reason` and `turns_completed`.

## Customize the pattern

1. Point `assistant_agent` and `user_agent` at any independently hosted
   Responses API agents.
2. Put assistant tools in `responses_create_params.tools` and user tools in
   `user_responses_create_params.tools`.
3. Implement `/episode_status` on the resources server. Return
   `{"terminated": bool, "reason": str | null, "state": {...}}`.
4. Keep shared mutable state on the resources server and access it through
   propagated session cookies.
5. Implement `/verify` against the final state and attributed trajectories.

Environment termination is authoritative. `max_turns` remains a bounded
fallback when the environment does not terminate naturally.
