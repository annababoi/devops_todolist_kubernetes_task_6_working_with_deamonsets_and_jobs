# DaemonSet and CronJob: Django ToDo App

## Prerequisites

- A running Kubernetes cluster and a configured `kubectl` client.
- The Django ToDo application is running in the `mateapp` namespace.
- A ClusterIP Service named `todoapp` exposes the application on port `80` in the same namespace.

Check the namespace, application pods, and Service before deploying:

```bash
kubectl get namespace mateapp
kubectl get pods -n mateapp
kubectl get svc todoapp -n mateapp
```

Both manifests use the `ikulyk404/busyboxplus:curl` container image and define CPU and memory resource requests and limits.

## 1. Deploy the DaemonSet

From the repository root, apply the DaemonSet manifest:

```bash
kubectl apply -f daemonset.yml
```

Check the rollout status and the DaemonSet:

```bash
kubectl rollout status daemonset/todolist -n mateapp
kubectl get daemonset todolist -n mateapp
kubectl get pods -n mateapp -l app=todolist -o wide
```

The DaemonSet should run one Pod on each eligible node. The `DESIRED`, `CURRENT`, and `READY` counts should match once the rollout is complete.

### Validate the DaemonSet

The container sends a curl request to the ToDo application's ClusterIP Service every five seconds (with a five-second sleep between requests):

```text
http://todoapp:80
```

Display recent logs from all DaemonSet pods:

```bash
kubectl logs -n mateapp -l app=todolist --all-containers=true --prefix=true --timestamps=true --tail=30
```

Follow the logs of a DaemonSet Pod to verify repeated requests:

```bash
kubectl logs -f daemonset/todolist -n mateapp --timestamps=true
```

Expected result: the logs show repeated curl requests and responses from the ToDo application, with a five-second pause between requests. Check that there are no DNS or connection errors. Press `Ctrl+C` to stop following the logs.

## 2. Deploy the CronJob

Apply the CronJob manifest:

```bash
kubectl apply -f cronjob.yml
```

Check the CronJob configuration:

```bash
kubectl get cronjob todolist-cronjob -n mateapp
kubectl describe cronjob todolist-cronjob -n mateapp
```

The CronJob is configured as follows:

| Setting | Value |
| --- | --- |
| Schedule | `*/4 * * * *` (every four minutes) |
| Concurrency policy | `Allow` |
| Successful Jobs history limit | `10` |
| Failed Jobs history limit | `5` |
| Starting deadline | `100` seconds |
| Restart policy | `OnFailure` |
| HTTP endpoint | `http://todoapp:80/api/health` |

### Validate the CronJob

Wait for the next scheduled execution and list the Jobs:

```bash
kubectl get jobs -n mateapp --watch
```

A Job with a name beginning with `todolist-cronjob-` should appear. Wait until the Job completes, then press `Ctrl+C` to stop watching.

List the Jobs and their pods:

```bash
kubectl get jobs -n mateapp
kubectl get pods -n mateapp
```

To view the logs of a scheduled run, replace `<JOB_NAME>` with the actual name of a completed Job created by the CronJob:

```bash
kubectl logs -n mateapp job/<JOB_NAME>
```

Expected result: the Job has `1/1` completions, and the logs contain the execution date and the health endpoint response, if it returns a body. The `curl -fsS` command reports HTTP errors (status 400 or higher) as failures.

Verify that additional Jobs are created every four minutes:

```bash
kubectl get cronjob todolist-cronjob -n mateapp
kubectl get jobs -n mateapp --sort-by=.metadata.creationTimestamp
```

### Optional: test the CronJob immediately

A manual Job can be created from the CronJob without waiting for the next scheduled run:

```bash
kubectl create job todolist-test --from=cronjob/todolist-cronjob -n mateapp
kubectl wait --for=condition=complete job/todolist-test -n mateapp --timeout=120s
kubectl logs -n mateapp job/todolist-test
```

If the Job does not complete, inspect its status and pod logs:

```bash
kubectl describe job todolist-test -n mateapp
kubectl get pods -n mateapp -l job-name=todolist-test
kubectl logs -n mateapp job/todolist-test
```

Remove the manual test Job when finished:

```bash
kubectl delete job todolist-test -n mateapp
```

If the endpoint returns a redirect, inspect the HTTP status and check the Django URL configuration (including whether `/api/health/` is required). A redirect alone does not prove that the health endpoint succeeded.

The manual test verifies the command and endpoint. A scheduled Job must also be checked to verify the actual CronJob schedule.

## 3. Validation checklist

- [ ] The DaemonSet is deployed in `mateapp`, and one Pod is ready on each eligible node.
- [ ] DaemonSet logs show repeated curl requests to the `todoapp` ClusterIP Service approximately every five seconds.
- [ ] The CronJob is deployed in `mateapp` and creates Jobs every four minutes.
- [ ] A scheduled Job completes successfully and its logs show the execution date and health-check result (if the response has a body).
- [ ] The CronJob uses `Allow`, a successful history limit of `10`, and a failed history limit of `5`.
- [ ] Both manifests specify CPU and memory requests and limits.

## 4. Collect validation logs

After a successful deployment, collect real log output for the submission. Run the following commands from the repository root:

```bash
kubectl logs -n mateapp -l app=todolist --all-containers=true --prefix=true --timestamps=true --tail=30 | tee daemonset-validation.log
kubectl logs -n mateapp job/<JOB_NAME> --timestamps=true | tee cronjob-validation.log
```

Replace `<JOB_NAME>` with the name of an actual completed, scheduled Job before running the second command. If the assignment requires log output directly in this document, paste the contents of these two log files below.

**DaemonSet logs:**

```text
Paste actual DaemonSet log output here after validation.
```

**CronJob logs:**

```text
Paste actual scheduled CronJob log output here after validation.
```

## 5. Cleanup (optional)

After validation, remove the resources if they are no longer needed:

```bash
kubectl delete -f cronjob.yml
kubectl delete -f daemonset.yml
```

This does not remove the existing ToDo application or its ClusterIP Service.
