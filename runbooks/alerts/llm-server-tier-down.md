# LlmServerTierDown

| Alert | Severity | Condition |
|-------|----------|-----------|
| LlmServerTierDown | warning | `probe_success{job="blackbox-llm-servertier"} == 0` for 10m |

**Signal**: a blackbox HTTP probe of the always-on CPU inference tier's health endpoint, probed directly on the guest, never through the LLM router (the router wakes the desktop fast tier on demand, which a probe must not do). The fast tier has a probe too but stays info-only by design: "the desktop is asleep" is a normal state.

## Triage

1. `curl -s http://<server>:8080/health` from the operations host.
2. On the guest: `systemctl status llama-swap`, `journalctl -u llama-swap -n 50`. A model swap or reload shows a short health gap; 10 minutes does not.
3. Guest down: on the hypervisor `qm status <id>`; check the host's memory headroom alerts, this guest holds a large resident model.
4. Nothing family-facing depends on this tier; the chat front end falls back to it when the desktop is in gaming mode, so the user-visible symptom is "slow or no answer in the chat app" only while the desktop is also unavailable.
