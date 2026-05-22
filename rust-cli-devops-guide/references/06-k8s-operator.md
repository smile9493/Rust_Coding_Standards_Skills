# 06 — Kubernetes Operator Pattern

## Status
**Approved.** Use `kube-rs` for writing Kubernetes operators and controllers in Rust. The reconciler loop pattern is mandatory for all operators.

## Prerequisites
- `cargo add kube --features derive,runtime,client`
- `cargo add k8s-openapi --features v1_30`
- `cargo add tokio --features full`
- `cargo add schemars` (for CRD schema generation)
- `cargo add thiserror`

---

## 1. Custom Resource Definition (CRD) via Derive Macro

```rust
use kube::CustomResource;
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

#[derive(CustomResource, Serialize, Deserialize, Debug, Clone, JsonSchema)]
#[kube(
    group = "devkit.io",
    version = "v1alpha1",
    kind = "Project",
    plural = "projects",
    singular = "project",
    shortname = "proj",
    namespaced,
    status = "ProjectStatus",
    printcolumn = r#"{
        "name": "Phase",
        "type": "string",
        "description": "Current phase of the project",
        "jsonPath": ".status.phase"
    }"#
)]
pub struct ProjectSpec {
    pub name: String,
    pub template: String,
    #[serde(default = "default_replicas")]
    pub replicas: i32,
    pub git_repo: Option<String>,
}

fn default_replicas() -> i32 {
    1
}

#[derive(Serialize, Deserialize, Debug, Clone, JsonSchema)]
pub struct ProjectStatus {
    pub phase: Option<String>,
    pub message: Option<String>,
    pub observed_generation: Option<i64>,
    #[serde(default)]
    pub conditions: Vec<k8s_openapi::apimachinery::pkg::apis::meta::v1::Condition>,
}
```

---

## 2. Reconciler Loop — Idempotent Pattern

```rust
use kube::{
    api::Api,
    runtime::controller::{Action, Controller},
    Client, ResourceExt,
};
use std::sync::Arc;
use tokio::time::Duration;

#[derive(thiserror::Error, Debug)]
enum OperatorError {
    #[error("Kube error: {0}")]
    Kube(#[from] kube::Error),
    #[error("Validation error: {0}")]
    Validation(String),
}

struct Context {
    client: Client,
}

async fn reconcile(
    project: Arc<Project>,
    ctx: Arc<Context>,
) -> Result<Action, OperatorError> {
    let api: Api<Project> = Api::namespaced(ctx.client.clone(), &project.namespace().unwrap());

    tracing::info!(
        "Reconciling project: {} (generation: {})",
        project.name_any(),
        project.metadata.generation.unwrap_or(0)
    );

    // Step 1: Validate spec
    validate_spec(&project.spec)?;

    // Step 2: Create or update ConfigMap
    apply_configmap(&ctx.client, &project).await?;

    // Step 3: Create or update Deployment
    apply_deployment(&ctx.client, &project).await?;

    // Step 4: Create or update Service
    apply_service(&ctx.client, &project).await?;

    // Step 5: Update status
    update_status(&api, &project, "Running", "All resources reconciled").await?;

    // Requeue after 30 seconds for next check
    Ok(Action::requeue(Duration::from_secs(30)))
}

fn validate_spec(spec: &ProjectSpec) -> Result<(), OperatorError> {
    if spec.name.is_empty() {
        return Err(OperatorError::Validation("spec.name must not be empty".into()));
    }
    if spec.replicas < 1 {
        return Err(OperatorError::Validation(
            "spec.replicas must be >= 1".into(),
        ));
    }
    Ok(())
}
```

---

## 3. Apply Resource — Idempotent Upsert

```rust
use kube::{
    api::{Api, Patch, PatchParams},
    core::ObjectMeta,
};
use k8s_openapi::api::apps::v1::{Deployment, DeploymentSpec};

async fn apply_deployment(
    client: &Client,
    project: &Project,
) -> Result<(), OperatorError> {
    let api: Api<Deployment> = Api::namespaced(
        client.clone(),
        &project.namespace().unwrap(),
    );

    let deployment = Deployment {
        metadata: ObjectMeta {
            name: Some(project.name_any()),
            namespace: Some(project.namespace().unwrap()),
            owner_references: Some(vec![project.controller_owner_ref(&()).unwrap()]),
            ..Default::default()
        },
        spec: Some(DeploymentSpec {
            replicas: Some(project.spec.replicas),
            ..Default::default()
        }),
        ..Default::default()
    };

    let pp = PatchParams::apply("devkit-operator").force();
    api.patch(&project.name_any(), &pp, &Patch::Apply(deployment))
        .await?;

    Ok(())
}
```

---

## 4. Finalizer — Prevent Orphaned Resources

```rust
const FINALIZER: &str = "devkit.io/finalizer";

async fn add_finalizer(api: &Api<Project>, project: &Project) -> Result<(), kube::Error> {
    if project.metadata.finalizers.as_ref().map_or(true, |f| !f.contains(&FINALIZER.to_string())) {
        let mut proj = project.clone();
        proj.metadata.finalizers = Some(vec![FINALIZER.to_string()]);
        api.patch(
            &project.name_any(),
            &PatchParams::default(),
            &Patch::Merge(&proj),
        )
        .await?;
    }
    Ok(())
}

async fn cleanup_external_resources(
    ctx: &Context,
    project: &Project,
) -> Result<(), OperatorError> {
    let api: Api<ConfigMap> = Api::namespaced(
        ctx.client.clone(),
        &project.namespace().unwrap(),
    );
    let name = format!("{}-config", project.name_any());
    if api.get_opt(&name).await?.is_some() {
        api.delete(&name, &Default::default()).await?;
        tracing::info!("cleaned up configmap: {name}");
    }
    Ok(())
}

async fn handle_deletion(
    api: &Api<Project>,
    project: &Project,
    ctx: &Context,
) -> Result<Action, OperatorError> {
    cleanup_external_resources(ctx, project).await?;

    let mut proj = project.clone();
    proj.metadata.finalizers = Some(
        project
            .metadata
            .finalizers
            .clone()
            .unwrap_or_default()
            .into_iter()
            .filter(|f| f != FINALIZER)
            .collect(),
    );
    api.patch(
        &project.name_any(),
        &PatchParams::default(),
        &Patch::Merge(&proj),
    )
    .await?;

    tracing::info!("finalizer removed for {}", project.name_any());
    Ok(Action::await_change())
}
```

---

## 5. Status Update with Conditions

```rust
use chrono::Utc;

async fn update_status(
    api: &Api<Project>,
    project: &Project,
    phase: &str,
    message: &str,
) -> Result<(), kube::Error> {
    let now = k8s_openapi::apimachinery::pkg::apis::meta::v1::Time(Utc::now());

    let condition = k8s_openapi::apimachinery::pkg::apis::meta::v1::Condition {
        type_: "Ready".to_string(),
        status: if phase == "Running" {
            "True".to_string()
        } else {
            "False".to_string()
        },
        last_transition_time: now,
        reason: "Reconciled".to_string(),
        message: message.to_string(),
        ..Default::default()
    };

    let status = serde_json::json!({
        "status": {
            "phase": phase,
            "message": message,
            "observedGeneration": project.metadata.generation,
            "conditions": [condition],
        }
    });

    api.patch_status(
        &project.name_any(),
        &PatchParams::default(),
        &Patch::Merge(&status),
    )
    .await?;
    Ok(())
}
```

---

## 6. Controller Bootstrap

```rust
use kube::runtime::watcher;

#[tokio::main]
async fn main() -> Result<(), OperatorError> {
    tracing_subscriber::fmt::init();

    let client = Client::try_default().await?;
    let ctx = Arc::new(Context { client: client.clone() });

    let projects: Api<Project> = Api::all(client.clone());

    tracing::info!("Starting devkit operator...");

    Controller::new(projects.clone(), watcher::Config::default())
        .run(reconcile, error_policy, ctx)
        .for_each(|res| async move {
            match res {
                Ok(_) => {}
                Err(e) => tracing::error!("controller error: {e}"),
            }
        })
        .await;

    Ok(())
}

fn error_policy(
    _obj: Arc<Project>,
    err: &OperatorError,
    _ctx: Arc<Context>,
) -> Action {
    tracing::error!("reconcile failed: {err}");
    Action::requeue(Duration::from_secs(60))
}
```

---

## Red Lines

| Rule | Rationale |
|------|-----------|
| **Mandatory**: reconciler must be idempotent | Running reconcile twice with the same input must produce the same cluster state. Use `Patch::Apply` (server-side apply) for all resources. |
| **Mandatory**: use finalizers for external resource cleanup | Without finalizers, `kubectl delete` orphans cloud resources (load balancers, buckets, DNS records). |
| **Mandatory**: `ownerReferences` on created resources | Kubernetes garbage collector automatically cleans up owned resources when the parent CR is deleted. |
| **Forbid** blocking IO in reconcile | Reconcile is async. Use `tokio::fs`, `reqwest`, or `kube` client. Never call `std::thread::sleep`. |
| **Forbid** unwrap in reconcile error path | All reconciliation failures must be `OperatorError`. Never `.unwrap()` — it crashes the entire controller. |

---

## References
- [kube-rs documentation](https://docs.rs/kube/latest/kube/)
- [kube-rs controller example](https://github.com/kube-rs/kube/blob/main/examples/crd_derive.rs)
- [Kubernetes controller guidelines](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md)