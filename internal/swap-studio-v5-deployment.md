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

## V3 face detection configuration

Configure `host.facedetectv3` in the effective OpenAPI environment configuration or Nacos document. Set the base URL of the service exposing `/detect_faces`, for example `https://<face-detection-service>`, without the endpoint path. This is required by Kling Character Swap reference/first-frame detection and Hair Swap. The legacy `host.facedetect` serves `/detect_multi` and is not a fallback for V3.

If detection fails, verify configuration injection, service connectivity, and the `[detect_face_v3]` trace ID, HTTP status and business error. Reload the service after configuration changes and retest the original single-person inputs. Do not place internal service addresses or credentials in public API pages.
## Motion Swap submission failure: MaaS configuration

For DEV bug `reczz28ND9z670xD`, Job logs identified `maasBackend configuration is required` behind the public `Swap submission failed` message. Configuration has been supplied according to the user; effective runtime loading and end-to-end generation remain unverified.

Configure the **open-api-job** environment/Nacos document:

```js
maasBackend: {
    api_url: 'https://<maas-service>/api/v1',
    api_key: '<injected-secret>',
}
```

Alternatively use `MAAS_BACKEND_API_URL` and `MAAS_BACKEND_API_KEY`. An existing `app.config.maasBackend` object takes precedence as a whole: an incomplete object does not fall back to environment variables field by field. Supply both fields or remove the object when relying on environment variables. Do not append `/run` to the base URL or store actual secrets in this document.

Routing follows the runtime provider and queued task: `maas`/`pixverse` use MaaS; `wavespeed-ai`/`wavespeed`/`kwaivgi` use WaveSpeed. A Kling public model ID may be routed through MaaS. This failure is unrelated to `host.facedetectv3`.

Reload/restart Job after configuration changes, create a new task, verify provider task ID assignment, final status, credit settlement, and webhook delivery. Previously failed tasks do not automatically resume; do not replay their queue messages without reconciling credit state. A log showing the failure settlement service was called does not by itself prove that credits were released. Retest each affected available PixVerse/Kling model and record the actual provider, task ID, sanitized logs, and ledger outcome.

## Prompt configuration precedence and algorithm routing

OpenAPI accepts `character_swap_prompt` (source AIGC naming) and `characterSwapPromptModel`. Fields resolve from `characterSwapPromptModel` to `character_swap_prompt` to `CHARACTER_SWAP_PROMPT_*` environment variables; `apiKey` and `api_key` are accepted. The model finally defaults to `gpt-5.5`. Timeout and other options merge from source configuration with the newer object taking precedence.

Analysis jobs use `algorithmFrom: characterSwap`, `algorithmType: analyze`, with task/queue `character_analyze` for character/head analysis or `sam3_analyzer` for clothing analysis. Clothing generation uses `algorithmFrom: clothSwap`, `algorithmType: clothSwap`, task/queue `cloth_swap`. Verify worker consumption and token-protected callback reachability when analysis stays waiting.

Kling character replacement resolves the configured `alias.motion_model` and checks its availability, account access and resolution before starting image editing. Runtime model documents determine the actual provider. Public responses include downstream `task_id`, which can be empty before submission or change across stages; use `_id` for API polling.

## Price parity verification (2026-10-16)

The local snapshot matches all 10 public models and the internal GPT image-edit model in the current AIGC source `unit_price_map`; character image fee constants match as well. This does not verify runtime Mongo documents. Clothing requires runtime model pricing and cannot be compared from its placeholder snapshot.

Final charges may differ: AIGC Hair Swap has a configured 0.66 membership discount which OpenAPI does not apply. OpenAPI resolves account feature pricing before team pricing and normal model prices. Character image replacement uses the low-quality edit tier plus 2 credits; hair multiple views use four times the base task quote. Clothing image generation uses each service's runtime `image_deduction_credit`. Compare actual account/team pricing and ledger outcomes before declaring production billing parity. Do not expose supplier costs in public model responses or documentation.
