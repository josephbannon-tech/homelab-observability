# Verify the alert path with a synthetic alert

**When**: after changing the notification webhook, the Alertmanager config, or route matchers. Also worth running on a schedule; a broken notification path is otherwise only discovered by a real outage arriving in silence.

**Why this shape**: the test injects at the Alertmanager API, which exercises routing, grouping, templating, and webhook delivery, i.e. everything downstream of Prometheus. The `endsAt` timestamp makes it self-cleaning: no leftover firing alert, and you get to verify resolved-message delivery for free.

```bash
# Port-forward Alertmanager (ClusterIP-only by design; the webhook URL is the
# secret, not the API).
kubectl -n monitoring port-forward svc/kube-prometheus-stack-alertmanager 9093:9093 >/dev/null 2>&1 &
PF=$!; sleep 2

# Fire a clearly-named alert with an explicit endsAt so it auto-resolves.
NOW=$(date -u +%Y-%m-%dT%H:%M:%S.000Z)
END=$(date -u -d '+5 minutes' +%Y-%m-%dT%H:%M:%S.000Z)
curl -sS -X POST -H "Content-Type: application/json" \
  -d "[{\"labels\":{\"alertname\":\"SyntheticTestAlert\",\"severity\":\"warning\",\"service\":\"synthetic\"},\"annotations\":{\"summary\":\"Synthetic test alert\",\"description\":\"Manual routing verification. Auto-resolves at endsAt.\"},\"startsAt\":\"$NOW\",\"endsAt\":\"$END\"}]" \
  http://127.0.0.1:9093/api/v2/alerts

# Confirm it registered:
curl -sS http://127.0.0.1:9093/api/v2/alerts | python3 -c \
  "import sys, json; [print(a['labels']['alertname'], a['status']['state']) for a in json.load(sys.stdin)]"

kill $PF 2>/dev/null
```

**Expected in the notification channel**:

- A **firing** message within ~10 seconds of the POST (`group_wait` plus webhook latency).
- A **resolved** message at `endsAt`. With the default `resolve_timeout` of 5 minutes, it can lag `endsAt` by up to that much.

## If nothing arrives within 60 seconds

Work down the path in order:

1. **Is the alert in Alertmanager?** `curl http://127.0.0.1:9093/api/v2/alerts` and look for it. If absent, the POST failed; check its response body.
2. **Did delivery fail?** `kubectl logs -n monitoring -l app.kubernetes.io/name=alertmanager | tail -60`, looking for 4xx/5xx from the webhook or DNS failures.
3. **Did routing match?** `kubectl exec -n monitoring <alertmanager-pod> -- amtool config routes test alertname=SyntheticTestAlert severity=warning`. The output should end at the intended receiver.
4. **Rotated webhook?** If the webhook URL changed, re-seal the secret that carries the Alertmanager config, push, let GitOps reconcile, and restart the Alertmanager pods to force a re-read.
