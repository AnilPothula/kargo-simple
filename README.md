# Kargo Simple Example

This is a GitOps repository of a simple Kargo example for getting started.

### Features:

* A Warehouse which monitors a container repository for new images
* Three Stage (dev, staging, prod) deploy pipeline
* Image tag promotion
* PromotionTask with a conditional pull request
* Direct Git commits to dev, staging
* Pull request for promotion to prod

This example does not require an Argo CD instance and so would work with any
GitOps operator (Argo CD, Flux) that detects and deploys manifest changes from
a path in a Git repository automatically (e.g. using auto-sync).

## Requirements

* Kargo v1.3.x (for older Kargo versions, switch to the release-X.Y branch)
* GitHub and a container registry (GHCR.io)
* `git` and `podman` installed

## Instructions

1. Fork this repo, then clone it locally (from your fork).
2. Run the `personalize.sh` to customize the manifests to use your GitHub
   username:

   ```shell
   ./personalize.sh <yourgithubusername>
   ```
3. `git commit` the personalized changes:

   ```shell
   git commit -a -m "personalize manifests"
   git push
   ```
4. Create a guestbook container image repository in your GitHub account. 

   The easiest way to create a new ghcr.io image repository, is by retagging and 
   pushing an existing image with your GitHub username:

   ```shell
   # https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-with-a-personal-access-token-classic
   echo $GHCR_PAT | podman login ghcr.io -u <yourgithubusername> --password-stdin

   podman manifest create ghcr.io/<yourgithubusername>/guestbook:v0.0.1
   podman manifest add ghcr.io/<yourgithubusername>/guestbook:v0.0.1 ghcr.io/akuity/guestbook:latest
   podman manifest push ghcr.io/<yourgithubusername>/guestbook:v0.0.1
   ```

   You will now have a `guestbook` container image repository. e.g.:
   https://github.com/users/yourgithubusername/packages/container/package/guestbook

5. Change guestbook container image repository to public.

   In the GitHub UI, navigate to the "guestbook" container repository, Package
   settings, and change the visibility of the package to public. This will allow
   Kargo to monitor this repository for new images, without requiring you to 
   configuring Kargo with container image repository credentials.

   ![change-package-visibility](docs/change-package-visibility.png)

6. Download and install the latest CLI from [Kargo Releases](https://github.com/akuity/kargo/releases/latest)

   ```shell
   sudo ./download-cli.sh /usr/local/bin/kargo
   ```

7. Login to Kargo:

   Install Kargo and login to your Kargo instance as an admin user
   ```sh
   # Generate a random password and signing key
   pass=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)
   echo "Password: $pass"
   hashed_pass=$(htpasswd -bnBC 10 "" $pass | tr -d ':\n')
   signing_key=$(openssl rand -base64 48 | tr -d "=+/" | head -c 32)

   # Install Kargo with Helm
   helm install kargo \
   oci://ghcr.io/akuity/kargo-charts/kargo \
   --namespace kargo \
   --create-namespace \
   --set-string api.adminAccount.passwordHash="$hashed_pass" \
   --set-string api.adminAccount.tokenSigningKey="$signing_key" \
   --set image.repository=ghcr.io/akuity/kargo \
   --wait

   # Forward the Kargo API port to localhost
   kubectl port-forward -n kargo svc/kargo-api 8080:443
   kargo login --admin https://localhost:8080 --insecure-skip-tls-verify --password $pass
   ```

8. Apply the Kargo manifests:

   ```shell
   kargo apply -f ./kargo
   ```

9. Add the Git repository credentials to Kargo. This can also be done in the UI
   in the `kargo-simple` Project.

   ```shell
   # When prompted for a token, use the GitHub Personal Access Token created earlier (GHCR_PAT)
   kargo create credentials github-creds \
     --project kargo-simple \
     --git \
     --username <yourgithubusername> \
     --repo-url https://github.com/<yourgithubusername>/kargo-simple.git
   ```

   As part of the promotion process, Kargo requires privileges to commit changes
   to your Git repository, as well as the ability to create pull requests. Ensure
   that the given token has these privileges.

10. Promote the image!

    You now have a Kargo Pipeline which promotes images from the guestbook
    container image repository, through a three-stage deploy pipeline. Visit
    the `kargo-simple` Project in the Kargo UI to see the deploy pipeline.

    ![pipeline](docs/pipeline.png)

    To promote, click the target icon to the left of the `dev` Stage, select
    the detected Freight, and click `Yes` to promote. Once promoted, the Freight
    will be qualified to be promoted to downstream Stages (`staging`, `prod`).


## Simulating a release

To simulate a release, simply retag an image with a newer semantic version. e.g.:

```shell
   podman manifest create ghcr.io/<yourgithubusername>/guestbook:v0.0.2
   podman manifest add ghcr.io/<yourgithubusername>/guestbook:v0.0.2 ghcr.io/akuity/guestbook:latest
   podman manifest push ghcr.io/<yourgithubusername>/guestbook:v0.0.2
```

Then refresh the Warehouse in the UI to detect the new Freight.
