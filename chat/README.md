# Chat (Open WebUI)

[Open WebUI](https://github.com/open-webui/open-webui) is an internal chat UI for testing
cluster models by hand. It is not the production front door: production traffic goes
through Bifrost directly ([`../bifrost/`](../bifrost/)). This UI is a convenience for
poking at models. It speaks the OpenAI API and points at **Bifrost** rather than at any
single model, so it automatically sees every model registered in the gateway.

Exposed externally at `chat.gd03.me`.

## Files

| File | What it is |
|------|-----------|
| `openwebui.yaml` | Deployment + Service + HTTPRoute, all in one |
| `httproute-openwebui.yaml` | A standalone HTTPRoute variant |

`openwebui.yaml` already includes an HTTPRoute (backend port 80).
`httproute-openwebui.yaml` is an alternate route (backend port 8080) kept for reference.
Apply the combined file.

## Install

```bash
kubectl apply -f openwebui.yaml
```

The config that matters (`openwebui.yaml`):

- `OPENAI_API_BASE_URL = http://bifrost.bifrost.svc.cluster.local/v1` points the UI at
  the Bifrost gateway over CoreDNS.
- `OPENAI_API_KEY = na`, since Bifrost handles the real auth/keys upstream.
- The Service is `ClusterIP` on port 80, targeting container port 8080.
- The HTTPRoute attaches to the `https-openwebui` listener on the cluster gateway for
  `chat.gd03.me`.

Once Bifrost has backends registered, they show up as selectable models in Open WebUI.
