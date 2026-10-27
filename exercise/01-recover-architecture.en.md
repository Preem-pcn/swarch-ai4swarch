**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| D1: Shopper → `frontend` | `frontend.yaml` (`Service frontend-external`, `type: LoadBalancer`, port `http`) | ✅ | `frontend.yaml` (lines 126–136): `frontend-external` exposes port 80 and targets container port 8080. |
| D5: `frontend` → `cartservice` | `frontend.yaml` (`CART_SERVICE_ADDR=cartservice:7070`); `cartservice.yaml` (port `grpc`) | ✅ | `frontend.yaml` (line 73): `CART_SERVICE_ADDR=cartservice:7070`. The protocol is inferred from the configuration. |
| D8: `frontend` → `checkoutservice` | `frontend.yaml` (`CHECKOUT_SERVICE_ADDR=checkoutservice:5050`); `checkoutservice.yaml` (port `grpc`) | ✅ | `frontend.yaml` (line 79): `CHECKOUT_SERVICE_ADDR=checkoutservice:5050`. |
| D13: `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`); `emailservice.yaml` (Service port 5000, name `grpc`) | ✅ | `checkoutservice.yaml` (line 63): `EMAIL_SERVICE_ADDR=emailservice:5000`. `emailservice.yaml` (lines 82–83) maps Service port 5000 to container port 8080. |
| D17: `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`; Service `redis-cart` port `tcp-redis`) | ✅ | `cartservice.yaml` (lines 50–51): `REDIS_ADDR=redis-cart:6379`; the Redis Service port is named `tcp-redis` (lines 154–156). |

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - Protocol evidence (D3–D16): gRPC is inferred from Service port names and health probes, which does not independently prove the protocol used by application calls.
  - Missing dependency: `frontend` → `shoppingassistantservice` is configured in `frontend.yaml` (`SHOPPING_ASSISTANT_SERVICE_ADDR`) but omitted from the dependency table (only mentioned under Gaps, G2). Its actual use is uncertain because the related component is commented out.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?

   Run A could identify the services and RPCs supported by the system from `demo.proto`, but it could not confirm which services call one another, which ports they use, or whether each service is deployed as a separate container. This is because `.proto` files define service interfaces rather than deployment configurations. Run B used Kubernetes manifests to identify more dependencies and port configurations, although some protocols and actual runtime interactions still require further verification.

2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?

   The most misleading mistake would be omitting `frontend` → `shoppingassistantservice` from the architecture diagram while listing it only under Gaps. A new team member might conclude that the frontend has no connection to the shopping assistant, even though its manifest defines `SHOPPING_ASSISTANT_SERVICE_ADDR`. However, this configuration alone does not confirm that the frontend actually calls the service at runtime, so the dependency should be marked as configured but not yet verified.
