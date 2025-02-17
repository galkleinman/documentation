---
id: patching
title: How to patch Workflow code in TypeScript
description: The TypeScript SDK Patching API lets you change Workflow Definitions without causing non-deterministic behavior in current long-running Workflows.
sidebar_label: Patching
tags:
  - typescript
  - workflow code
  - patching
  - versioning
---

The TypeScript SDK Patching API lets you change Workflow Definitions without causing [non-deterministic behavior](/concepts/what-is-a-workflow-definition#non-deterministic-change) in current long-running Workflows.

### Do I need to Patch?

You may need to patch if:

- You want to change the remaining logic of a Workflow while it is still running
- If your new logic can result in a different execution path

This added `sleep()` can result in a different execution path:

```ts
// from v1
export async function yourWorkflow(value: number): Promise<number> {
  await runActivity();
  return 7;
}

// to v2
export async function yourWorkflow(value: number): Promise<number> {
  await sleep('1 day');

  await runActivity();
  return 7;
}
```

If v2 is deployed while there's a Workflow on the `runActivity` step, when the Activity completes, the Worker will try to replay the Workflow (in order to continue Workflow execution), notice that the sleep command is called and doesn't match with the Workflow's Event History, and throw a non-determinism error.

Adding a Signal Handler for a Signal type that has never been sent before does not need patching:

```ts
// from v1
export async function yourWorkflow(value: number): Promise<number> {
  await sleep('1 days');
  return value;
}

// to v2
const updateValueSignal = defineSignal<[number]>('updateValue');

export async function yourWorkflow(value: number): Promise<number> {
  setHandler(updateValueSignal, (newValue) => (value = newValue));

  await sleep('1 days');
  return value;
}
```

### Migrating Workflows in Patches

Workflow code has to be [deterministic](/workflows#deterministic-constraints) by taking the same code path when replaying History Events.
Any Workflow code change that affects the order in which commands are generated breaks this assumption.

So we have to keep both the old and new code when migrating Workflows while they are still running:

- When replaying, use the original code version that generated the ongoing event history.
- When executing a new code path, always execute the
  new code.

<details>
<summary>30 Min Video: Introduction to Versioning
</summary>

Because we design for potentially long-running Workflows at scale, versioning with Temporal works differently than with other Workflow systems.
We explain more in this optional 30 minute introduction: [https://www.youtube.com/watch?v=kkP899WxgzY](https://www.youtube.com/watch?v=kkP899WxgzY)

</details>

### TypeScript SDK Patching API

In principle, the TypeScript SDK's patching mechanism works in a similar "feature-flag" fashion to the other SDKs; however, the "versioning" API has been updated to a notion of "patching in" code.
There are three steps to this reflecting three stages of migration:

- Running v1 code with vFinal patched in concurrently
- Running vFinal code with deprecation markers for vFinal patches
- Running "just" vFinal code.

This is best explained in sequence (click through to follow along using our SDK sample).

Given an initial Workflow version `v1`:

<!--SNIPSTART typescript-patching-1-->

[patching-api/src/workflows-v1.ts](https://github.com/temporalio/samples-typescript/blob/master/patching-api/src/workflows-v1.ts)

```ts
// v1
export async function myWorkflow(): Promise<void> {
  await activityA();
  await sleep('1 days'); // arbitrary long sleep to simulate a long running workflow we need to patch
  await activityThatMustRunAfterA();
}
```

<!--SNIPEND-->

We decide to update our code and run `activityB` instead.
This is our desired end state, `vFinal`.

<!--SNIPSTART typescript-patching-final-->

[patching-api/src/workflows-vFinal.ts](https://github.com/temporalio/samples-typescript/blob/master/patching-api/src/workflows-vFinal.ts)

```ts
// vFinal
export async function myWorkflow(): Promise<void> {
  await activityB();
  await sleep('1 days');
}
```

<!--SNIPEND-->

**Problem: We cannot directly deploy `vFinal` until we know for sure there are no more running Workflows created using `v1` code.**

Instead we must deploy `v2` (below) and use the [`patched`](https://typescript.temporal.io/api/namespaces/workflow#patched) function to check which version of the code should be executed.

Patching is a three-step process:

1. Patch in new code with `patched` and run it alongside old code
2. Remove old code and `deprecatePatch`
3. When you are sure all old Workflows are done executing, remove `deprecatePatch`

#### Step 1: Patch in new code

`patched` inserts a marker into the Workflow history.

![image](https://user-images.githubusercontent.com/6764957/139673361-35d61b38-ab94-401e-ae7b-feaa52eae8c6.png)

During replay, when a Worker picks up a history with that marker it will fail the Workflow task when running Workflow code that does not emit the same patch marker (in this case `your-change-id`); therefore it is safe to deploy code from `v2` in a "feature flag" alongside the original version (`v1`).

<!--SNIPSTART typescript-patching-2-->

[patching-api/src/workflows-v2.ts](https://github.com/temporalio/samples-typescript/blob/master/patching-api/src/workflows-v2.ts)

```ts
// v2
import { patched } from '@temporalio/workflow';
export async function myWorkflow(): Promise<void> {
  if (patched('my-change-id')) {
    await activityB();
    await sleep('1 days');
  } else {
    await activityA();
    await sleep('1 days');
    await activityThatMustRunAfterA();
  }
}
```

<!--SNIPEND-->

#### Step 2: Deprecate patch

When we know that all Workflows started with `v1` code have completed, we can [deprecate the patch](https://typescript.temporal.io/api/namespaces/workflow#deprecatepatch).
Deprecated patches bridge between `v2` and `vFinal` (the end result).
They work similarly to regular patches by recording a marker in the Workflow history.
This marker does not fail replay when Workflow code does not emit it.

If while we're deploying `v3` (below) there are still live Workers running `v2` code and those Workers pick up Workflow histories generated by `v3`, they will safely use the patched branch.

<!--SNIPSTART typescript-patching-3-->

[patching-api/src/workflows-v3.ts](https://github.com/temporalio/samples-typescript/blob/master/patching-api/src/workflows-v3.ts)

```ts
// v3
import { deprecatePatch } from '@temporalio/workflow';

export async function myWorkflow(): Promise<void> {
  deprecatePatch('my-change-id');
  await activityB();
  await sleep('1 days');
}
```

<!--SNIPEND-->

#### Step 3: Solely deploy new code

`vFinal` is safe to deploy once all `v2` or earlier Workflows are complete due to the assertion mentioned above.

### Upgrading Workflow dependencies

Upgrading Workflow dependencies (such as ones installed into `node_modules`) _might_ break determinism in unpredictable ways.
We recommended using a lock file (`package-lock.json` or `yarn.lock`) to fix Workflow dependency versions and gain control of when they're updated.
