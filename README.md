# NOAA Alerts

## Overview

As a means of learning Kubernetes, and _possibly_ preparing to take the CKAD, I've been tinkering with [K3s](https://k3s.io/) running on a 4-node Raspberry Pi cluster I built.

`noaa-alerts` combines several new K8s topics with rough implementation of Simon Willison's ["git scraping"](https://simonwillison.net/2020/Oct/9/git-scraping/?utm_source=pocket_saves) technique. I chose the National Oceanic and Atmospheric Administration's (NOAA) API as source data because Hurricane Hillary was set to hit Southern California and I was already looking at the updates via various UIs.

## K8s Resources

To scrape a subset of the alerts for California from NOAA's API, which can be found at `https://api.weather.gov/openapi.json`), I used a [ConfigMap](https://kubernetes.io/docs/concepts/configuration/configmap/) for the shell script and a [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/) for the scheduled job.

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: git-scrape-configmap
data:
  git-scrape.sh: |
    #!/bin/sh
    echo "Creating repo  directory ..."
    mkdir /noaa-alerts && cd /noaa-alerts
    echo "Initializing git repo and setting username and email ..."
    git config --global init.defaultBranch main
    git init
    git config user.email "orca@crime.com"
    git config user.name "willythekid"
    git remote add origin https://<PERSONAL ACCESS TOKEN SCOPED TO THIS REPO>@github.com/epps/noaa-alerts.git/
    git pull origin main
    echo "Fetch latest CA alerts and saving to ca-alerts.json ..."
    curl https://api.weather.gov/alerts/active/area/CA | jq '.features[].properties | {id, headline, event, severity, certainty, urgency, status, areaDesc}' > ca-alerts.json
    echo "Adding ca-alerts.json to git repo ..."
    git add .
    timestamp=$(date -u)
    git commit -m "CA NOAA alerts as of $timestamp"
    echo "Pushing to remote ..."
    git push -u origin main
    echo "Committed ca-alerts.json to git repo; job complete!"
```

### CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: noaa-alerts-cron
  labels:
    app: noaa-alerts
spec:
  schedule: "*/12 * * * *"
  startingDeadlineSeconds: 60
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: noaa-ca-alerts
              image: epps/git-scraping:v1
              imagePullPolicy: Always
              command: ["/bin/sh", "/usr/local/bin/git-scrape.sh"]
              volumeMounts:
                - name: script
                  mountPath: "/usr/local/bin"
          volumes:
            - name: script
              configMap:
                name: git-scrape-configmap
                defaultMode: 0500
          restartPolicy: Never
```

## Challenges and Learning

Despite the simplicity of this little project, I bumped into many issues that forced me to consider and troubleshoot details that don't normally surface in my day-to-day work (e.g. user permissions on the shell script added by the ConfigMap). I hope to expand on this and polish it into something more mature and full-featured.
