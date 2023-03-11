# action-upload-to-firebase-storage
GitHub Actions to upload build/test artifacts (e.g. HTML test reports) to Firebase Storage with OIDC.

- Upload HTML reports to a Google Cloud Storage bucket.
- Uses [OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform) — no secret keys or service account involved.

## Why?

- When tests are run, they generate **HTML reports.**

- I want to upload the HTML reports to where it can be easily viewed without having to download first.

- I have many projects, and I don’t want to manage the secret key in each project.

  - [GitHub Actions supports OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect) which is [also supported by Google Cloud Platform](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-google-cloud-platform). It allows GitHub Actions to securely access the cloud service (e.g. Firebase Storage bucket) without having to store a secret key on GitHub Actions side.

## Usage

```yaml
      - uses: dtinth/action-upload-to-firebase-storage@main
        if: always()
        continue-on-error: true
        id: publish
        with:
          workload-identity-provider: 'projects/__/locations/global/workloadIdentityPools/__/providers/__'
          service-account: '__@__.iam.gserviceaccount.com'
          storage-prefix: 'gs://__.appspot.com'
          path: 'playwright-report'
      - run: |
          if [ -n "$REPORT_URL" ]; then echo "# [View test report]($REPORT_URL)" > $GITHUB_STEP_SUMMARY; fi
        env:
          REPORT_URL: ${{ steps.publish.outputs.http-url }}/index.html
        if: always()
        continue-on-error: true
```

## Setting up

Before using this action you have to set up a Firebase and Google Cloud project to store the files. This is a one-time setup and the resulting setup can be re-used across multiple repositories.

1. **Create a Firebase project** to host the storage bucket.
2. **Create a service account** to access cloud resources.
3. **Create an identity pool** to recognize GitHub Actions as a trusted entity.
4. **Allow GitHub Actions to impersonate the service account.**
5. **Allow the service account to access the storage bucket.**
6. **Allow public access** to the storage bucket.
7. **Set up lifecycle rules** to automatically delete old files.

Finally, you can [Set up GitHub Actions](#usage) to upload build artifacts to the storage bucket.

### Create a Firebase Project

1. Create a Firebase project.
2. Go to **Storage** and select a region to activate the storage bucket.

### Access the Google Cloud Console

The Firebase Console has limited functionality. But every Firebase project comes with a Google Cloud Platform project, which lets you do more advanced stuff with the free resources you get from Firebase.

1. Go to <https://console.cloud.google.com/> to access the Google Cloud Platform console.
2. Select the Firebase project.

### Create a service account

1. Go to **IAM & Admin** > **Service Accounts**.
2. Click **Create Service Account**.
3. Type in the service account name and click **Done**. No need to grant any roles or create a key.
    - Roles will be granted at the storage bucket.
    - The service account will be impersonated by GitHub Actions, so no key is needed.
4. Take note of the **Email** — it will be used later.

### Create Workload Identity Pool

1. Go to **IAM & Admin** > **Workload Identity Federation**.
2. Click **Create Pool** (or **Get Started**).
3. Create an identity pool.
    - **Name**: `GitHub Actions`
    - **Pool ID**: `github-actions`
    - Click **Continue**.
4. Add a provider to the pool.
    - **Select a provider**: `OIDC`
    - **Provider name**: `GitHub Actions`
    - **Issuer URL**: `https://tokens.actions.githubusercontent.com`
    - Take note of the **Default audience** value — it will be used later.
    - Click **Continue**.
5. Configure provider attributes.
    - Set up attribute mapping:
        - `google.subject` &larr; `assertion.sub`
        - `attribute.actor` &larr; `assertion.actor`
        - `attribute.repository` &larr; `assertion.repository`
    - To restrict to your own repositories, add a condition:
        - `attribute.repository.startsWith("your-username/")`
    - Click **Save**.

### Allow GitHub Actions to impersonate the service account

1. Go to the Identity Pool you created in the previous step.
2. Click on **Grant Access**.
    - **Select service account**: (Select the service account you created earlier.)
3. Click **Save**.

### Allow the service account to access the storage bucket

1. Go to **Cloud Storage** > **Buckets**.
2. Click the default storage bucket.
3. Go to **Permissions** tab.
4. Click **Grant Access**.
    - **New principal**: (Enter the service account email you created earlier.)
    - **Role**: `Storage Object Admin`

### Allow public access to the storage bucket

1. Go to **Cloud Storage** > **Buckets**.
2. Click the default storage bucket.
3. Go to **Permissions** tab.
4. Click **Grant Access**.
    - **New principal**: `allUsers`
    - **Role**: `Storage Object Viewer`

### Set up lifecycle rules to automatically delete old files

1. Go to **Cloud Storage** > **Buckets**.
2. Click the default storage bucket.
3. Go to **Lifecycle** tab.
4. Click **Add a Rule**.
    - **Select an action:** `Delete object`
    - Click **Continue**.
5. Select object conditions
    - **Set conditions:**
        - **Age**: `90` days
6. Click **Create**.
