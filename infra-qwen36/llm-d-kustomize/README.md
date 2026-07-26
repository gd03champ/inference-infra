# llm-d kustomize overlay (Qwen3.6-27B-FP8)

Copy these files into the (gitignored) upstream llm-d clone at:

```
src/llm-d/guides/optimized-baseline/modelserver/gpu/vllm/qwen-3.6/
```

The `../` paths inside `kustomization.yaml` are relative to that location. They resolve
into `guides/recipes/...`, so the overlay only works from inside the guides tree.

- `kustomization.yaml` composes the base model server, the `gpu-vllm` image component,
  and the `monitoring` component, applies the name prefix and model labels, and pulls in
  the patch.
- `patch-vllm.yaml` patches the `decode` Deployment: `baseline-qwen36` scheduling, the
  `vllm serve` args (MTP=3, `max-num-seqs=32`, 128k context), GPU resources, the HF
  token, cache volumes, and probes.

Apply:

```bash
kubectl apply -n infra-qwen36 -k src/llm-d/guides/optimized-baseline/modelserver/gpu/vllm/qwen-3.6/
```

See [`../README.md`](../README.md) for the full deployment runbook and the tuning story.
