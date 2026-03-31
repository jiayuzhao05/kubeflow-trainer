# Dynamic LLM Trainer Framework

**Problem:**
Kubeflow Trainer currently supports LLM fine-tuning exclusively through TorchTune as its only built-in backend. TorchTune-specific logic is tightly coupled into the Torch plugin, the BuiltinTrainer SDK pattern, and the trainer container images. Since TorchTune is no longer actively adding new features, this single-backend, the architecture makes Kubeflow cannot easily support emerging models or increasingly popular post-training methods such as DPO, PPO, and ORPO. Every time the community wants to adopt a new fine-tuning framework, it requires invasive changes to the core Torch plugin and SDK client, rather than a clean extension.



**Solution:**
We propose a dynamic LLM Trainer Framework that decouples Kubeflow Trainer from any single fine-tuning backend by introducing a pluggable, backend-agnostic architecture. The design builds on the existing runtime extension framework plugin registry in pkg/runtime/framework/) and the BuiltinTrainer SDK pattern, and extends them to support dynamic backend registration for both in-tree and external frameworks.
We can define a backend-agnostic LLM Trainer interface, and the Torch plugin delegates to a selected backend rather than hardcoding TorchTune behavior. Then build a registry mechanism on both the control plane and the Python SDK, enabling multiple backends to coexist and be selected at TrainJob submission time. And migrate existing TorchTune functionality into the new framework with full backward compatibility, Finally set TRL as a second backend through demonstrating extensibility by adding TRL support which enables SFT/DPO/PPO fine-tuning workflows.



**Deliverables:**
A dynamic backend registration mechanism for both Go plugins and the Python SDK, supporting in-tree and external framework registration.TorchTune refactored as a pluggable backend, fully backward compatible with existing TrainJob and ClusterTrainingRuntime workflows. TRL implemented as a new pluggable backend with SFT/DPO support, including ClusterTrainingRuntime templates and container images. And unit tests for each backend module, integration tests for the registry and plugin pipeline.

## Personal Information

- **Full Name**: jiayuzhao
- **Email Address**:jiayuzhao@uchicago.edu
- **GitHub Profile**: https://github.com/jiayuzhao05
- **LinkedIn**: https://www.linkedin.com/in/fenney-zhao-74642a1b9/
- **University/College**: university of chicago
- **Timezone**: utc-5



### Contributions to Kubeflow

- **kubeflow/trainer**: Added Trivy security scanning documentation to the project README ([`docs-trivy-security-note` branch](https://github.com/kubeflow/trainer)).
- **kubeflow/trainer#3368**: Reviewed the MPI SSH auth secret volume permission fix. Identified a behavioral risk where a single `DefaultMode` of 0640 applies to the entire volume — including private keys — which can trigger SSH "permissions too open" errors. Recommended per-item modes instead of a volume-wide `DefaultMode` to enforce least-privilege.
- **kubeflow/trainer#3366**: Reviewed the flaky E2E notebook completion fix. Identified that while `timeout=20` was applied to specific notebooks, it was not uniformly updated across all notebooks, meaning the underlying flakiness was not systematically resolved.
- **kubeflow/pipelines#13089**: Found a user-visible regression in the `NewRunV2` `mutateAsync` refactor. The `isRecurringRun` state could change between `try` and `catch` blocks due to async operations, causing incorrect error dialog titles. Suggested capturing the submitted mode before the first `await`.
- **kubeflow/pipelines#13069**: Identified a blocking regression in the multi-arch docker buildx cherry-pick. The `build-and-push.yml` still exposes a standalone `workflow_dispatch` entrypoint while the build step now only pushes anonymous digest references, meaning manual runs no longer publish requested target/latest tags.
- **kubeflow/pipelines#13065**: Reviewed the pod lifecycle failure reason UI PR. Noted it is intentionally a partial frontend fix and that backend extraction from `containerStatuses` requires separate tracking.



## Goals

1. **Design and implement a backend-agnostic LLM Trainer interface** that abstracts fine-tuning backend behavior (validation, command construction, environment mutation) behind a clean Go interface.
2. **Implement a dynamic backend registry** on both the Go control plane and the Python SDK, allowing backends to register themselves and be selected at TrainJob submission time.
3. **Refactor TorchTune into the first pluggable backend** with zero regression to existing users, proving the abstraction's soundness.
4. **Implement TRL as a second pluggable backend**, enabling SFT and DPO fine-tuning workflows with full `ClusterTrainingRuntime` support.
5. **Deliver comprehensive tests** (unit + integration) for each backend and the registry mechanism.
6. **Write contributor documentation** explaining how to add new external backends.



### Architecture Overview

The current flow for LLM fine-tuning in Kubeflow Trainer is TrainJob (spec.trainer)

→ TrainingRuntime (ClusterTrainingRuntime with TorchTune image)

→ runtime.Info (carries template + ML policy)

→ Torch plugin EnforceMLPolicy (hardcoded TorchTune logic in torchtune.go)

→ JobSet plugin Build → Kubernetes objects (JobSet, Services)



The proposed architecture introduces a backend interface between the Torch plugin and framework-specific logic TrainJob

→ TrainingRuntime (ClusterTrainingRuntime with backend-specific image)

→ runtime.Info

→ Torch plugin EnforceMLPolicy

→ Backend Registry → selected Backend (TorchTune | TRL | ...)

→ Backend.Validate(), Backend.BuildCommand(), Backend.InjectEnv()

→ JobSet plugin Build → Kubernetes objects



**Implement Plan**

### Phase 1: Backend Interface

Define a Go interface in `pkg/runtime/framework/plugins/torch/`

Extract the existing TorchTune functions (validateTorchTune, getRecipeAndConfig, extractOverridesFromRuntime) from torchtune.go into a struct implementing this interface. Modify torch.go's EnforceMLPolicy to delegate to the selected backend via the registry.



### Phase 2: Backend Registry

Go side: Implement a backend registry that maps backend names to constructors, similar to the existing plugin registry pattern in registry.go

Python SDK side: Extend the BuiltinTrainer pattern to accept a backend parameter



### Phase 3: TorchTune Refactoring (Weeks 5–7)

Migrate all TorchTune-specific logic from torchtune.go into pkg/runtime/framework/plugins/torch/backends/torchtune/

Recipe/config resolution, multi-node rendezvous, QLoRA validation all move into the backend

The torch.go EnforceMLPolicy becomes framework-agnostic, delegating to backend.BuildCommand() and backend.InjectEnv()

All existing tests in torch_test.go must continue to pass without modification

### Phase 4: TRL Backend

Implement pkg/runtime/framework/plugins/torch/backends/trl/:

backend.go: Implements LLMTrainerBackend for TRL, supporting SFT and DPO training methods

cmd/trainers/trl/Dockerfile: Container image with TRL, transformers, peft, and accelerate



## Test Plan

### Unit Tests (Go)

Backend interface tests: Verify each backend's Validate, BuildCommand, InjectEnv, and ExtractOverrides methods using table-driven test cases, following the existing pattern in torch_test.go.

Registry tests: Verify registration, lookup, duplicate registration error handling, and unknown backend error handling.

Refactored Torch plugin tests: All existing tests in torch_test.go must pass without modification after the refactoring, serving as regression tests.

TRL backend tests: Method-specific validation (SFT, DPO), command construction for single-node and multi-node, environment variable injection.

### Unit Tests (Python)

SDK backend registration tests: Verify BuiltinTrainer correctly selects backends and generates the expected TrainJob spec.



### Integration Tests

For Ginkgo integration tests in test/integration/, verify end-to-end flow from TrainJob creation through runtime resolution, backend selection, plugin pipeline execution, to JobSet creation for both TorchTune and TRL backends.

For backward compatibility testsm, submit existing TorchTune-based TrainJob specs and verify the generated JobSet is identical to the pre-refactoring output.