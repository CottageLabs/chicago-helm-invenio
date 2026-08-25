# Monitoring and alerting

This document covers the CloudWatch-based monitoring and alerting set up for
the `chicago-invenio` EKS cluster.

## Background

On 2026-08-25, a worker node's kubelet stopped responding while the
underlying EC2 instance continued to report healthy to AWS (all EC2 status
checks passed). Because AWS's own health checks never saw a problem, nothing
auto-remediated it, and the outage (RabbitMQ down, two of three OpenSearch
masters gone) went unnoticed until someone happened to run `kubectl get
pods`. This setup closes that gap: it alerts on the *Kubernetes-level*
signals (node conditions, pod restarts) that AWS's own infrastructure
monitoring doesn't cover.

See also [`../charts/invenio/values-uchicago.yaml`](../values-uchicago.yaml)
`opensearch.antiAffinity: "hard"`, added at the same time to stop OpenSearch
master pods from being able to land on the same node in the first place.

---

## Container Insights (metrics + log collection)

Installed via the `amazon-cloudwatch-observability` EKS add-on, which
deploys the CloudWatch agent and Fluent Bit as DaemonSets (one pod per
node) and publishes metrics to the `ContainerInsights` CloudWatch namespace.

### 1. Associate an OIDC provider

Required for IAM Roles for Service Accounts (IRSA). Safe to run even if
already associated.

```bash
AWS_PROFILE=<your-profile> eksctl utils associate-iam-oidc-provider \
  --config-file=cluster.yaml --approve
```

### 2. Create the IAM role for the agent

Declared in `cluster.yaml` under `iam.serviceAccounts` (`roleOnly: true` —
this creates only the IAM role, not a Kubernetes ServiceAccount, so the
add-on can create its own ServiceAccount without a naming conflict):

```yaml
iam:
  withOIDC: true
  serviceAccounts:
    - metadata:
        name: cloudwatch-agent
        namespace: amazon-cloudwatch
      attachPolicyARNs:
        - "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
      roleName: chicago-invenio-cloudwatch-agent-role
      roleOnly: true
```

Apply it:

```bash
AWS_PROFILE=<your-profile> eksctl create iamserviceaccount \
  --config-file=cluster.yaml --approve
```

### 3. Install the add-on

This is installed imperatively rather than declared in `cluster.yaml`'s
`addons:` list, so the account ID doesn't end up hardcoded in a committed
file — `AWS_ACCOUNT` is resolved at run time instead.

```bash
AWS_ACCOUNT=$(AWS_PROFILE=<your-profile> aws sts get-caller-identity --query Account --output text)

AWS_PROFILE=<your-profile> aws eks create-addon \
  --addon-name amazon-cloudwatch-observability \
  --cluster-name chicago-invenio \
  --service-account-role-arn arn:aws:iam::${AWS_ACCOUNT}:role/chicago-invenio-cloudwatch-agent-role \
  --region us-east-2
```

Wait for it to become `ACTIVE`:

```bash
AWS_PROFILE=<your-profile> aws eks describe-addon \
  --cluster-name chicago-invenio \
  --addon-name amazon-cloudwatch-observability \
  --region us-east-2 \
  --query "addon.status"
```

### 4. Verify

```bash
kubectl get pods -n amazon-cloudwatch
```

Expect one `cloudwatch-agent-*` and one `fluent-bit-*` pod per node, plus
the `amazon-cloudwatch-observability-controller-manager` pod.

Logs land in CloudWatch Logs under
`/aws/containerinsights/chicago-invenio/{application,dataplane,host}`.
Metrics land in the `ContainerInsights` namespace — see the [full metric
reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-metrics-EKS.html)
for what's available beyond the two alarms below.

Note this also enables CloudWatch Application Signals (APM) by default,
which we are not currently using or configured for.

---

## Alerting

### 1. SNS topic

```bash
AWS_PROFILE=<your-profile> aws sns create-topic \
  --name chicago-invenio-alerts --region us-east-2
```

Subscribe a recipient (each new subscriber must confirm via a link AWS
emails them before they'll receive anything):

```bash
AWS_PROFILE=<your-profile> aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-2:${AWS_ACCOUNT}:chicago-invenio-alerts \
  --protocol email \
  --notification-endpoint sysadmin+chicago@cottagelabs.com \
  --region us-east-2
```

### 2. Node failure alarm

Fires when any node has a failed condition (`NotReady`, `Unreachable`,
etc. — the same signal that would have caught 2026-08-25 immediately
instead of it going unnoticed).

```bash
AWS_PROFILE=<your-profile> aws cloudwatch put-metric-alarm \
  --alarm-name "chicago-invenio-node-failure" \
  --alarm-description "One or more EKS worker nodes in chicago-invenio have a failed condition (e.g. NotReady/Unreachable)" \
  --namespace ContainerInsights \
  --metric-name cluster_failed_node_count \
  --dimensions Name=ClusterName,Value=chicago-invenio \
  --statistic Maximum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-2:${AWS_ACCOUNT}:chicago-invenio-alerts \
  --ok-actions arn:aws:sns:us-east-2:${AWS_ACCOUNT}:chicago-invenio-alerts \
  --region us-east-2
```

### 3. Pod crash-loop alarm

`pod_number_of_container_restarts` is reported per pod *name*, which
churns on every redeploy, and is a cumulative counter that never resets —
so a plain threshold alarm on it would either miss pods entirely after a
redeploy or get stuck permanently in ALARM after one crash-loop episode.
Instead this uses a Metrics Insights query to aggregate restarts by
namespace, wrapped in `RATE()` so it alarms on *new* restarts happening
now, not the lifetime total:

```bash
cat > /tmp/pod-restart-metrics.json <<'EOF'
[
  {
    "Id": "q1",
    "Expression": "SELECT SUM(pod_number_of_container_restarts) FROM SCHEMA(\"ContainerInsights\", ClusterName,Namespace) WHERE ClusterName = 'chicago-invenio' AND Namespace = 'invenio'",
    "Period": 300,
    "ReturnData": false
  },
  {
    "Id": "e1",
    "Expression": "RATE(q1)",
    "Label": "invenio namespace pod restart rate",
    "Period": 300,
    "ReturnData": true
  }
]
EOF

AWS_PROFILE=<your-profile> aws cloudwatch put-metric-alarm \
  --alarm-name "chicago-invenio-pod-crashlooping" \
  --alarm-description "Pods in the invenio namespace are restarting repeatedly (proxy for CrashLoopBackOff)" \
  --metrics file:///tmp/pod-restart-metrics.json \
  --evaluation-periods 2 \
  --datapoints-to-alarm 2 \
  --threshold 0 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-2:${AWS_ACCOUNT}:chicago-invenio-alerts \
  --ok-actions arn:aws:sns:us-east-2:${AWS_ACCOUNT}:chicago-invenio-alerts \
  --region us-east-2
```

**Tuning note:** the threshold (any restart activity sustained across two
5-minute periods) is a first guess, not validated against real traffic. A
normal rolling deploy causes a brief one-off restart per pod too, which
could trip this if timed unluckily. If it's noisy in practice, raise
`--threshold` or `--datapoints-to-alarm`, or add more evaluation periods.

### Verify

```bash
AWS_PROFILE=<your-profile> aws cloudwatch describe-alarms \
  --alarm-names chicago-invenio-node-failure chicago-invenio-pod-crashlooping \
  --region us-east-2 \
  --query "MetricAlarms[].[AlarmName,StateValue]" --output text
```

New alarms start in `INSUFFICIENT_DATA` until the first metric datapoint
arrives (a few minutes), then settle into `OK`.

---

## Known gaps / follow-ups not yet implemented

- **RabbitMQ has no anti-affinity fix** — it runs as a single replica, so
  no node-spread setting protects it; any node it lands on failing takes
  the broker down entirely. A real fix would mean running a multi-node
  RabbitMQ cluster (quorum queues), which is a bigger architectural change.
- **No automated node remediation** — recovering from a node stuck
  `NotReady`/`Unreachable` while AWS reports it healthy still requires a
  human to notice the alarm, diagnose, and terminate the EC2 instance
  manually so the ASG replaces it. Kubernetes 1.28+'s
  `node.kubernetes.io/out-of-service` taint (or a controller that applies
  it automatically once it confirms via the AWS API that an unresponsive
  node's instance is actually gone) would remove the manual step.
