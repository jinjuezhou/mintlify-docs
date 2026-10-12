# Swap Studio V5 deployment notes

This file is an internal deployment reference and is not registered in the public documentation navigation.

## Services and dependencies

Deploy the corresponding OpenAPI and OpenAPI Job changes together. Both services require their existing MongoDB/OpenAPI model connection and Redis/Bull configuration. OpenAPI reads source model documents through its existing AIGC connection (`aimodelconfigs`); file changes alone do not update runtime model documents.

The new named queue is `task_swap_studio_v5_queue`. Ensure the Job consumer and the V5 polling/settlement schedule are loaded. Provider processing and customer callback delivery use the existing provider and webhook queues.

Algorithm-backed tasks require the existing algorithm queue endpoint and workers for `character_analyze`, `sam3_analyzer`, `swap_generate`, `cloth_swap`, and `hair_swap`. `host.openapi` must be the reachable public OpenAPI base URL so algorithm callbacks can reach the generated token-protected V5 callback URLs. Customer applications must not call these internal callback URLs.

## Configuration placeholders

Supply secrets through the deployment secret mechanism. The following values are placeholders only:

```dotenv
CHARACTER_SWAP_PROMPT_API_URL=https://<prompt-service>/v1
CHARACTER_SWAP_PROMPT_API_KEY=<PROMPT_API_KEY>
CHARACTER_SWAP_PROMPT_MODEL=<PROMPT_MODEL_ID>
MAAS_BACKEND_API_URL=https://<maas-service>/api/v1
MAAS_BACKEND_API_KEY=<MAAS_API_KEY>
```

OpenAPI also accepts the existing `characterSwapPromptModel` object with `url`, `api_key`, and `model`. Job accepts the existing `maasBackend` object with `api_url` and `api_key`. When those objects are present, they take precedence over the environment fallback. Existing `waveSpeedAi` configuration remains required for WaveSpeed-backed models.

Character image replacement and Kling character replacement require the prompt service and GPT image-edit model configuration. Hair multiple-view generation requires the configured MaaS image-edit provider and its model mapping, in addition to the primary hair algorithm.

## Runtime model configuration

Review added/updated model documents and their feature identity, provider mapping, supported parameters, priority, and pricing before import. Clothing Swap requires its source runtime model configuration; a missing configuration hides that model from `/models` and blocks generation. Do not run a replace-all import without reviewing the full replacement set.

Call each V5 `/models` endpoint with an authorized API key and verify `data.list`, supported case-sensitive resolutions, and configured `unit_price_map`. Runtime availability depends on enabled model documents and account permissions.

## Verification

1. Confirm old V4 endpoints retain their existing behavior.
2. Exercise V5 authentication, model restrictions, parameter failures, and user/team task isolation.
3. Run one task per feature through create, queue/algorithm submission, callback or polling, credit settlement, and final task detail.
4. Verify a completed analysis has `analysis_result.track_manifest_url` and person IDs, and reject mappings from another URL or analysis.
5. For Hair Swap, test single-person references, multi-person mappings, and multiple views (`high_45`, `side`, `back`).
6. Verify failed generation releases the prelocked credits and customer callbacks follow the existing signed/encrypted format.

If credit settlement reaches the `processing` or `error` state after a partial service failure, reconcile against the existing credit ledger before retrying settlement. The worker intentionally avoids automatically repeating an uncertain balance mutation.

Deploying files, importing model configuration, and validating a live provider task are separate steps; report each result separately.
