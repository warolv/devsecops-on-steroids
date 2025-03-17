## Detect CVEs in Built Images Using Trivy and GitLab CI

![devsecops-on-steroids](images/trivy-cves/0.png)

[This tutorial on my blog](https://igorzhivilo.com/2025/03/12/trivy-cves/)

This tutorial focuses on detecting CVEs in built images using Trivy and GitLab CI.

You’ll learn how to integrate Trivy into your GitLab pipeline to detect critical vulnerabilities. I’ll provide code examples demonstrating how we use it at Cynerio and show you how to send Slack notifications when a CVE is detected.

#### What is Trivy?

**Trivy** is an open-source security scanner developed by **Aqua Security**. It is widely used to detect **vulnerabilities (CVEs)**, **misconfigurations**, **secrets**, and **compliance issues** in:

* Container images (Docker, Kubernetes)
* Filesystems & repositories
* Infrastructure-as-Code (IaC) (Terraform, Kubernetes, Helm)
* SBOM (Software Bill of Materials)

#### Why is Trivy Useful?

* ✅ **Comprehensive Scanning** – Detects CVEs, secrets, and misconfigurations in multiple formats.
* ✅ **Fast & Lightweight** – Scans efficiently with a minimal performance impact.
* ✅ **CI/CD Integration** – Easily integrates with **GitLab CI/CD, Jenkins, GitHub Actions, and more**.
* ✅ **Shifts Security Left** – Identifies vulnerabilities early in the **development pipeline**.
* ✅ **Supports Multiple Sources** – Works with Docker images, OCI registries, Git repositories, and more.


#### How to Add Trivy to Your GitLab Pipeline?

Here’s how you can add Trivy to your .gitlab-ci.yml file:

##### Example: Scan Docker Images
```yaml
stages:
  - security_scan

trivy_scan:
  image:
    name: aquasec/trivy:latest
    entrypoint: [""]  # Ensures shell compatibility
  stage: security_scan
  script:
    - trivy image --exit-code 1 --severity HIGH,CRITICAL my-docker-image:latest
  allow_failure: false
```

##### Environment Variables (Optional)

You can use environment variables to configure Trivy:
* TRIVY_IGNORE_UNFIXED=true – Ignore vulnerabilities with no fixes available.
* TRIVY_SEVERITY=HIGH,CRITICAL – Only report high/critical vulnerabilities.
* TRIVY_CACHE_DIR=/trivy-cache – Define a cache directory.
* TRIVY_EXIT_CODE=1 – Fail the pipeline if vulnerabilities are found.

Optional Enhancements
* Report Output: Store scan results as an artifact for later review.
* Dependency Scanning: Use trivy fs to scan dependencies for vulnerabilities.

#### How We Use Trivy at Cynerio

##### Our pipeline consists of two main stages:

* **Build Stage**: During this stage, we build multiple container images.
* **Test Stage**: In this stage, we run various tests against all the images built in the previous stage.

Trivy is executed during the **test stage**, after the build stage, to scan the generated images for CVEs (Common Vulnerabilities and Exposures).

##### Adding code to .gitlab-ci.yml to execute a Trivy scan on multiple images.
```yaml
trivy_image_scan:
  image: 
    name: aquasec/trivy:latest
  script:
    - .ci/run-trivy-tests.sh
  cache:
    paths:
      - .trivycache/
  artifacts:
    reports:
      container_scanning: gl-container-scanning-report.json
```

All Trivy-related code has been moved to an external script, run-trivy-tests.sh, located in the .ci folder.

##### code of run-trivy-tests.sh
```bash
declare -a arr=("image1"
                "image2" 
                "image3")

touch gl-container-scanning-report.json
for image in "${arr[@]}"
do
    echo "IMAGE NAME: $image\n"
    # Build report
    ./trivy image --ignore-unfixed --scanners vuln --exit-code 0 --cache-dir .trivycache/ --no-progress --severity CRITICAL --format template --template "@contrib/gitlab.tpl" -o $image-scan-report.json $REGISTRY/$image:$CI_COMMIT_SHORT_SHA
    # Print report
    ./trivy image --ignore-unfixed --scanners vuln --exit-code 0 --cache-dir .trivycache/ --no-progress --severity CRITICAL $REGISTRY/$image:$CI_COMMIT_SHORT_SHA
done

cat *-scan-report.json > gl-container-scanning-report.json
```

##### Notes
* Trivy scans multiple containers specified in the list: ['container1', 'container2', 'container3'].
* A separate image-scan-report.json is generated for each container.
* The scan focuses only on **critical** vulnerabilities (--severity CRITICAL) currently (next phase will be to add HIGH vulnerabilities also)
* **Unfixed vulnerabilities** are ignored using --ignore-unfixed.
* The pipeline **does not fail** if a critical CVE is detected (--exit-code 0).
* All individual reports are aggregated into a final report, **gl-container-scanning-report.json**, which is included as an artifact.
* $REGISTRY follows the format: '123456789.dkr.ecr.region.amazonaws.com'.
* $image:$CI_COMMIT_SHORT_SHA represents the built version of the image from the previous stage.

> It’s important to note that the **build does not fail if a critical CVE is found**. Instead, a notification is sent to the relevant Slack channel, and a critical issue is created for the appropriate teams.

#### Sending a Slack notification when a critical CVE is detected

To send a notification to Slack, we can use curl, requiring only the appropriate WEBHOOK_URL for the target channel

##### To Set Up a Slack Webhook
* Go to your Slack workspace and create an Incoming Webhook:
* Open Slack → Apps → Incoming Webhooks (Slack Webhook Setup)
* Create a new webhook and choose a Slack channel where alerts will be sent.
* Copy the Webhook URL, which looks like:

```bash
curl -X POST -H 'Content-type: application/json' --data '{"text":"Trivy Critical CVEs found in Branch: TEST_CVES_BRANCH"}' WEBHOOK_URL
```

> The webhook URL follows this format: https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX

Let's modify run-trivy-tests.sh to send notification to slack if critical CVE is detected
```bash
declare -a arr=("image1"
                "image2" 
                "image3")

touch gl-container-scanning-report.json
for image in "${arr[@]}"
do
    echo "IMAGE NAME: $image\n"
    # Build report
    ./trivy image --ignore-unfixed --scanners vuln --exit-code 0 --cache-dir .trivycache/ --no-progress --severity CRITICAL --format template --template "@contrib/gitlab.tpl" -o $image-scan-report.json $REGISTRY/$image:$CI_COMMIT_SHORT_SHA
    # Print report
    ./trivy image --ignore-unfixed --scanners vuln --exit-code 0 --cache-dir .trivycache/ --no-progress --severity CRITICAL $REGISTRY/$image:$CI_COMMIT_SHORT_SHA
done

cat *-scan-report.json > gl-container-scanning-report.json

### SEND SLACK NOTIFICATION if critical CVE is found
export WEBHOOK_URL=https://hooks.slack.com/services/T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX
export BRANCH_NAME=$CI_COMMIT_REF_NAME

cat gl-container-scanning-report.json | jq '.vulnerabilities | length' > vulns-results.txt

if grep -qE '^[1-9]' vulns-results.txt; then
  echo "Found Critical CVEs in Branch: $BRANCH_NAME \n"
  curl -X POST -H 'Content-type: application/json' --data '{"text":"Trivy Critical CVEs found in Branch: https://gitlab.com/cynerio/cloud/-/tree/'$BRANCH_NAME'"}' $WEBHOOK_URL
else
  echo "Not Found Critical CVEs!"
fi
```

##### Notes
* We use jq to populate vulns-results.txt with the number of vulnerabilities in each row, each row represent number of critical CVEs found for each container (In our example will be 3 rows). If any row contains a value greater than 0, a Slack notification will be sent.
* You must define $WEBHOOK_URL for slack channel to send notification


Thank you for reading, I hope you enjoyed it, see you in the next post.

Please subscribe to my [YT channel](https://www.youtube.com/@igorzhivilo) and [twitter](https://twitter.com/warolv), to be notified when the next tutorial is published.

You can find it also in my [blog](https://igorzhivilo.com/2025/03/12/trivy-cves/)
