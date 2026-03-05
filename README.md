# Lab Note: Build & Push Java Artifact to GitHub Packages using Maven + Jenkins Freestyle

---

## Prerequisites

- Jenkins instance with the following plugins installed: **Maven Integration**, **GitHub**, **Credentials Binding**
- JDK and Maven configured in Jenkins → Manage Jenkins → Global Tool Configuration
- A Java Maven project hosted on GitHub
- A GitHub Personal Access Token (PAT) with `write:packages` and `read:packages` scopes

---

## Step 1 — Configure GitHub PAT in Jenkins Credentials

1. Go to **Jenkins → Manage Jenkins → Credentials → (global) → Add Credentials**
2. Fill in:
   - **Kind:** `Username with password`
   - **Username:** your GitHub username
   - **Password:** your GitHub PAT
   - **ID:** `github-packages-creds` *(you'll reference this later)*
3. Click **Save**

---

## Step 2 — Update `pom.xml` for GitHub Packages

Add the distribution management block so Maven knows where to publish:

```xml
<distributionManagement>
  <repository>
    <id>github</id>
    <name>GitHub Packages</name>
    <url>https://maven.pkg.github.com/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME</url>
  </repository>
</distributionManagement>
```

Ensure your `pom.xml` has the correct artifact identifiers:

```xml
<groupId>com.yourcompany</groupId>
<artifactId>your-artifact</artifactId>
<version>1.0.0</version>
<packaging>jar</packaging>
```

> **Note:** The `<id>github</id>` here must exactly match the server `id` in `settings.xml` (Step 3).

---

## Step 3 — Create `settings.xml` on the Jenkins Server

Maven needs credentials at deploy time. Create the file at the Jenkins user's home directory:

```bash
mkdir -p /var/lib/jenkins/.m2
nano /var/lib/jenkins/.m2/settings.xml
```

Paste in the following content:

```xml
<settings>
  <servers>
    <server>
      <id>github</id>
      <username>${env.GITHUB_USERNAME}</username>
      <password>${env.GITHUB_TOKEN}</password>
    </server>
  </servers>
</settings>
```

Verify the file exists and is correctly named:

```bash
ls /var/lib/jenkins/.m2/
# Expected output: settings.xml
```

> **Common mistake:** Watch for filename typos like `settinga.xml`. Maven will throw a "file does not exist" error if the name is wrong. Always verify with `ls`.

---

## Step 4 — Create a Jenkins Freestyle Job

1. Go to **Jenkins → New Item → Freestyle Project**, give it a name, click **OK**

2. **Source Code Management:**
   - Select **Git**
   - Enter your repo URL: `https://github.com/YOUR_USERNAME/YOUR_REPO.git`
   - Add credentials — select `github-packages-creds`
   - Set branch to `*/main` (or your target branch)

3. **Build Triggers** *(optional but recommended):*
   - **GitHub hook trigger for GITScm polling** — for webhook-based triggers, or
   - **Poll SCM** — use cron like `H/5 * * * *`

---

## Step 5 — Bind Credentials as Environment Variables

1. Under **Build Environment**, check **Use secret text(s) or file(s)**
2. Click **Add → Username and password (separated)**
3. Fill in:
   - **Username Variable:** `GITHUB_USERNAME`
   - **Password Variable:** `GITHUB_TOKEN`
   - **Credentials:** select `github-packages-creds`

This injects your PAT safely as environment variables during the build so `settings.xml` can reference them.

---

## Step 6 — Configure the Build Step

1. Under **Build**, click **Add build step → Invoke top-level Maven targets**
2. Set:
   - **Maven Version:** select the Maven installation from Global Tools
   - **Goals:**
     ```
     clean deploy -s /var/lib/jenkins/.m2/settings.xml
     ```

> **Important:** Always use the full absolute path `/var/lib/jenkins/.m2/settings.xml`.  
> Do **not** use `~/.m2/settings.xml` or `$HOME/.m2/settings.xml` in the Maven targets field —  
> the tilde and variables are not shell-expanded in this build step and will cause a path error.

To skip tests:
```
clean deploy -DskipTests -s /var/lib/jenkins/.m2/settings.xml
```

---

## Step 7 — Save and Run the Job

1. Click **Save**, then **Build Now**
2. Monitor **Console Output** — a successful deploy will show:

```
[INFO] BUILD SUCCESS
```

Along with upload confirmation lines like:

```
Uploading to github: https://maven.pkg.github.com/YOUR_USERNAME/YOUR_REPO/...
Uploaded to github: https://maven.pkg.github.com/...
```

---

## Step 8 — Verify on GitHub

1. Navigate to your GitHub repository
2. Click **Packages** (right sidebar or top navigation)
3. Your artifact should be listed with the version defined in `pom.xml`

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `The specified user settings file does not exist` | Wrong path or filename typo (e.g. `settinga.xml`) | Run `ls /var/lib/jenkins/.m2/` and rename with `mv settinga.xml settings.xml` |
| Path resolves to workspace (e.g. `/var/lib/jenkins/workspace/deploy/~/.m2/...`) | Using `~` in Maven targets field — tilde is not expanded | Use full absolute path `/var/lib/jenkins/.m2/settings.xml` |
| `401 Unauthorized` | PAT missing `write:packages` scope or wrong credentials bound | Regenerate PAT with correct scopes; verify credential ID matches |
| `server id mismatch` | `id` in `settings.xml` doesn't match `id` in `pom.xml` `<distributionManagement>` | Ensure both are exactly `github` (or whatever you chose) |
| `409 Conflict` — artifact already exists | GitHub Packages does not allow overwriting a published version | Bump the version in `pom.xml` |
| Build succeeds but package not visible | Wrong repo URL in `<distributionManagement>` | Double-check `YOUR_GITHUB_USERNAME` and `YOUR_REPO_NAME` in the URL |

---

## Key Reminders

- The `-s` flag in **Invoke top-level Maven targets** does **not** perform shell expansion. Always use the full absolute path.
- The `<id>` tag in `settings.xml` and `pom.xml` must be **identical**.
- GitHub Packages is **immutable** — you cannot overwrite an existing version. Always increment your version for each publish.
- If unsure of the Jenkins home directory, run: `getent passwd jenkins | cut -d: -f6`
