# Cron Job Examples

Hermes Agent supports scheduled autonomous tasks via cron jobs. Examples:

## Daily NVIDIA Forum Summary
```
Schedule: 0 9 * * *
Prompt: Search the last 24h of NVIDIA forum posts,
        summarize top 3 discussions, flag any DGX Spark issues.
Skills: [nvidia-forum]
Deliver: telegram
```

## Weekly Infrastructure Report  
```
Schedule: 0 10 * * MON
Prompt: Query Prometheus for CPU/memory/disk trends across
        all nodes. Generate a weekly health report.
Skills: [mac-monitoring, k8s-monitoring]
Deliver: telegram
```

## Hourly Service Health Check
```
Schedule: 30m
Prompt: Check all services (Forgejo, Grafana, n8n, Pi-hole).
        Alert if any are down.
Deliver: telegram
```

## Automated Code Review
```
Schedule: every 2h
Prompt: Check for new PRs in flux-config and k8s-homelab repos.
        Run security scan, validate YAML syntax, flag issues.
Skills: [github-code-review, github-pr-workflow]
Deliver: telegram
```
