# DaemonSet and CronJob: Django ToDo App

## Prerequisites

- A running Kubernetes cluster with `kubectl` configured.
- The Django ToDo application is running in the `mateapp` namespace.
- A ClusterIP Service named `todoapp` in `mateapp` exposes the application on port `80`.
- The `busyboxplus:curl` image is available to the cluster.

Run the commands in this document from the repository root.

```bash
kubectl get namespace mateapp
kubectl get pods -n mateapp
kubectl get svc todoapp -n mateapp
kubectl get endpointslices -n mateapp -l kubernetes.io/service-name=todoapp
```

If the ToDo application is not running yet, deploy its existing manifests and wait for the application to become ready:

```bash
kubectl apply -f .infrastructure/namespace.yml
kubectl apply -f .infrastructure/deployment.yml
kubectl apply -f .infrastructure/clusterIp.yml
```

## 1. Deploy and validate the DaemonSet

```bash
kubectl apply -f .infrastructure/daemonset.yml
kubectl rollout status daemonset/todolist -n mateapp --timeout=180s
kubectl get daemonset todolist -n mateapp
kubectl get pods -n mateapp -l app=todolist -o wide
```

The DaemonSet runs one Pod on each eligible node. Its container uses `busyboxplus:curl`, defines CPU and memory requests and limits, and sends repeated HTTP requests to `http://todoapp:80`, with a five-second pause between requests.

Check that the DaemonSet has the expected number of ready Pods and review the logs from its Pods:

```bash
kubectl logs -n mateapp -l app=todolist --prefix=true --timestamps=true --tail=40
```

The logs should show repeated successful responses from the ToDo application. To follow the output live:

```bash
kubectl logs -n mateapp -l app=todolist --prefix=true --timestamps=true --follow --max-log-requests=20
```

Press Ctrl+C to stop following the logs.

## 2. Deploy and validate the CronJob

```bash
kubectl apply -f .infrastructure/cronjob.yml
kubectl get cronjob todolist-cronjob -n mateapp
kubectl describe cronjob todolist-cronjob -n mateapp
```

The CronJob configuration is:

| Setting | Value |
| --- | --- |
| Namespace | `mateapp` |
| Container image | `busyboxplus:curl` |
| Schedule | `*/4 * * * *` |
| HTTP endpoint | `http://todoapp:80/api/health` |
| Concurrency policy | `Allow` |
| Successful Jobs history limit | `10` |
| Failed Jobs history limit | `5` |
| Restart policy | `OnFailure` |

CPU and memory requests and limits are defined in the manifest. The `curl -fsS` command returns a non-zero exit code on HTTP errors (4xx/5xx) or connection failures.

Wait for a scheduled execution (up to four minutes until the next scheduled time), then check the Jobs:

```bash
kubectl get jobs -n mateapp --sort-by=.metadata.creationTimestamp
kubectl get pods -n mateapp
```

A successfully completed scheduled Job should have `1/1` in the COMPLETIONS column. Its name starts with `todolist-cronjob-`.

To view the logs, replace `JOB_NAME` below with the actual name of a completed scheduled Job:

```bash
kubectl logs -n mateapp job/JOB_NAME --timestamps=true
```

The logs show the execution date and the health endpoint response body, if one is returned. A completed Job confirms that the command exited successfully. The scheduled Job must be used as evidence; a manually triggered Job is not a substitute for this check.

## 3. Record actual validation logs

The following commands automatically select the latest successfully completed scheduled Job created by `todolist-cronjob` with the required image. They collect real DaemonSet and Job logs from the current cluster and append them to `INSTRUCTION.md`. Python 3 is required for selecting the Job. Run the complete block once from the repository root after applying both manifests and waiting for a successful scheduled run.

```bash
(
  set -e
  kubectl rollout status daemonset/todolist -n mateapp --timeout=180s
  JOB=$(kubectl get jobs -n mateapp -o json | python3 -c 'import json,sys; jobs=[j for j in json.load(sys.stdin)["items"] if any(o.get("kind")=="CronJob" and o.get("name")=="todolist-cronjob" for o in j["metadata"].get("ownerReferences",[])) and any(c.get("type")=="Complete" and c.get("status")=="True" for c in j.get("status",{}).get("conditions",[])) and any(c.get("image")=="busyboxplus:curl" for c in j["spec"]["template"]["spec"]["containers"])]; print(max(jobs,key=lambda j:j["metadata"]["creationTimestamp"])["metadata"]["name"] if jobs else "")')
  if [ -z "$JOB" ]; then
    echo "No completed scheduled Job with the required image yet. Wait for the next scheduled run and retry." >&2
    exit 1
  fi
  kubectl wait --for=condition=complete "job/$JOB" -n mateapp --timeout=60s
  DS_LOGS=$(kubectl logs -n mateapp -l app=todolist --prefix=true --timestamps=true --since=25s --tail=-1)
  JOB_LOGS=$(kubectl logs -n mateapp "job/$JOB" --timestamps=true)
  if [ -z "$DS_LOGS" ] || [ -z "$JOB_LOGS" ]; then
    echo "Logs are empty. Check the Pods and retry." >&2
    exit 1
  fi
  {
    printf '\n## 5. Actual validation logs\n\n'
    printf '### DaemonSet: todolist\n\n'
    printf '%s\n' '```text' "$DS_LOGS" '```'
    printf '\n### Scheduled CronJob: %s\n\n' "$JOB"
    printf '%s\n' '```text' "$JOB_LOGS" '```'
  } >> INSTRUCTION.md
  echo "Actual validation logs appended to INSTRUCTION.md"
)
```

Review the captured output to confirm successful HTTP responses and the completed scheduled Job before committing the file.

## 4. Cleanup (optional)

```bash
kubectl delete -f .infrastructure/daemonset.yml
kubectl delete -f .infrastructure/cronjob.yml
```

This removes the DaemonSet and CronJob without removing the existing ToDo application or its ClusterIP Service.
