# Phase 1 — Comparison Report

Copy this file to `submission/phase1_report.md`, fill in every answer slot, and commit it:

```bash
cp phase-1-echo-bot/report_template.md submission/phase1_report.md
```

**How to fill it in.** Each answer sits between a pair of `<!--answer:...-->` markers. Replace
the `TODO` line with your answer — **leave the markers themselves untouched**, they are how the
grader finds your answers. Anything you write outside the markers is ignored, so add extra
prose, tables or images freely.

Numbers must **match `submission/phase1_report.json`** exactly — that file is produced by
`measure.py` and is what your figures are checked against. Copy them across; do not retype from
memory.

---

## 1. Setup facts

**Your image size**, in MB, from `gcloud artifacts docker images list` (number only, e.g. `142`):

<!--answer:image_size_mb-->
48 Mo
<!--/answer-->

**The custom VPC and subnet you created** (names only):

<!--answer:vpc_name-->
echo-net
<!--/answer-->

<!--answer:subnet_name-->
echo-subnet
<!--/answer-->

---

## 2. Measurements

Copy these from `submission/phase1_report.json`. Cold is `cold_ms`; warm median is
`stats.median`. Numbers only, in milliseconds (e.g. `812.4`).

| Platform | Cold (ms) | Warm median (ms) |
|---|---|---|
| IaaS VM | <!--answer:vm_cold_ms-->250.0<!--/answer--> | <!--answer:vm_warm_median_ms-->210.8<!--/answer--> |
| Cloud Run | <!--answer:cloudrun_cold_ms-->657.7<!--/answer--> | <!--answer:cloudrun_warm_median_ms-->633.2<!--/answer--> |
| Cloud Function | <!--answer:function_cold_ms-->689.1<!--/answer--> | <!--answer:function_warm_median_ms-->632.8<!--/answer--> |

Include your histogram or plot of the warm distribution below (an image, or the ASCII chart
`measure.py` prints). This is read by a human, not the grader:

<!--answer:latency_plot-->
[OK  ] vm        cold=  250.0ms  warm_median=  210.8ms
            #############
[OK  ] cloudrun  cold=  657.7ms  warm_median=  633.2ms
            ########################################
[OK  ] function  cold=  689.1ms  warm_median=  632.8ms
            ########################################
<!--/answer-->

---

## 3. Per-platform analysis

**Scaling behaviour.** Imagine requests arriving one at a time, then ten at once, then a
hundred. For each of the three platforms, answer concretely:

- Does anything get created automatically to absorb the load — and if so, *what* exactly?
  (One platform here creates nothing at all.)
- What is the hard limit, and where does it come from — a flag you passed yourself, or the
  size of the machine you picked?

(~80 words. Your `--machine-type=e2-micro` and `--max-instances=3` are both concrete answers
to the second question.)

<!--answer:scaling_behaviour-->
- **Compute Engine (VM):** Creates nothing automatically when traffic spikes. The single `e2-micro` instance handles all incoming requests on its own. Its hard limit comes directly from the fixed hardware capacity (vCPU and RAM) of the `e2-micro` machine type.
- **Cloud Run:** Automatically provisions new container instances to handle concurrent traffic spikes. Its hard limit is capped at 3 instances, coming directly from the `--max-instances=3` flag set during deployment.
- **Cloud Function:** Automatically spins up new function instances per request spike. Its hard limit is governed by GCP project-level quotas or default concurrency limits on gen-2 functions.
<!--/answer-->

**Deployment effort.** What did you have to build, configure and maintain to get each of the
three running? (~100 words)

<!--answer:deployment_effort-->
- **Compute Engine (VM):** Highest effort. Requires building a custom VPC network, subnets, firewall rules, creating the instance, SSHing in, installing Docker, configuring IAM registry authentication, and manually managing the container lifecycle.
- **Cloud Run:** Moderate effort. Requires writing a Dockerfile, building the container image, pushing it to Artifact Registry, and running a single deployment command. Infrastructure is fully managed.
- **Cloud Function:** Lowest effort. Requires no Dockerfile or image management. Google handles the container build and runtime environment directly from the raw Python source code (`app.py` and `requirements.txt`).
<!--/answer-->

---

## 4. Why do two of your three numbers look alike?

With Cloud Run and your Function both deployed, run `gcloud run services list` and paste the
**complete output** here, unedited:

<!--answer:run_services_list-->
SERVICE: echo
REGION: us-central1
URL: https://echo-538366421711.us-central1.run.app
LAST DEPLOYED BY: service-538366421711@gcf-admin-robot.iam.gserviceaccount.com
LAST DEPLOYED AT: 2026-10-24T14:56:09.778341Z

✔
SERVICE: echo-bot
REGION: us-central1
URL: https://echo-bot-538366421711.us-central1.run.app
LAST DEPLOYED BY: gaston.rousset@eurecom.fr
LAST DEPLOYED AT: 2026-10-24T14:48:20.284692Z
<!--/answer-->

Now explain what that output tells you about how your Function is actually being run, why its
cold start resembles Cloud Run's so closely, and what — if anything — genuinely differs between
the two deployment paths. (~120 words)

<!--answer:convergence_explanation-->
The output shows that both the `echo-bot` Cloud Run service and the `echo` Cloud Function are listed as active Cloud Run services. This demonstrates that GCP 2nd-generation Cloud Functions run directly on top of Cloud Run infrastructure under the hood. 

When deploying a gen-2 function, GCP automatically uses Cloud Build and Buildpacks to package the source code into a Docker image and deploys it as a Cloud Run revision. Because both services run on identical containerized infrastructure, their cold-start provisioning times and warm execution latencies are virtually indistinguishable. The only real difference lies in developer workflow: Cloud Run expects a pre-built Docker image, whereas Cloud Functions abstracts the containerization process entirely by building directly from raw source files.
<!--/answer-->
