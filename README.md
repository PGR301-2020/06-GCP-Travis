# Integrasjon med travis og GCP


## Bli kjent med Google Clud Run tjenesten

Bruk en eksisterende REST Spring Boot mikrotjeneste. Fiks Dockerfile slik at den bygges som multi stage. Dere kan bruke mitt "ping pong" som eksempel ; https://github.com/PGR301-2020/01-devops-helloworld/

Eksmpel som bør kunne brukes direkte 
```
FROM maven:3.6-jdk-11 as builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn package 

FROM openjdk:8-jdk-alpine
COPY --from=builder /app/target/*.jar /app/application.jar
ENTRYPOINT ["java","-jar","/app/application.jar"]
```

Det er viktig at applikasjonen er konfigurert for å lytte på en port som bestemmes av miljøvariabelen PORTLek med tjenesten "Cloud Run" i GCP. Lag en ny "Service" og deploy rett fra repositoriet ved hjelp av Wizarden og GitHub integrasjon  i GCP Web-grensesnittet. Denne videoen viser deployment rett fra GitHub til Cloud Run. https://www.youtube.com/watch?v=GhSAQ19f4HA&ab_channel=KevinSimper

## Integrasjon med Travis

Disse instruksjonene er skamløst klippet fra https://github.com/PGR301-2020/cloud-run-travisci/blob/master/README.md

## Step 1: Install required tools

- Google Cloud SDK (`gcloud`): https://cloud.google.com/sdk

- `travis` command-line tool:

    ```sh
    sudo gem install travis
    ```

    ```sh
    travis login --pro # (use --org if you're on travis-ci.ORG and not .COM)
    ```

## Step 2: Create a service account for deploying

To authenticate to GCP APIs from Travis CI build environment you will need a
[service
account](https://cloud.google.com/iam/docs/understanding-service-accounts).

```sh
PROJECT_ID="$(gcloud config get-value project -q)" # fetch current GCP project ID
```

```sh
SVCACCT_NAME=travisci-deployer # choose name for service account
```

Create a service account:

```sh
gcloud iam service-accounts create "${SVCACCT_NAME?}"
```

Find the email address of this account:

```sh
SVCACCT_EMAIL="$(gcloud iam service-accounts list \
  --filter="name:${SVCACCT_NAME?}@"  \
  --format=value\(email\))"
```

Create a JSON key to authenticate as this service account, and save it as
`google-key.json`:

```sh
gcloud iam service-accounts keys create "google-key.json" \
   --iam-account="${SVCACCT_EMAIL?}"
```

## Step 3: Assign permissions to the service account

You need to give these IAM roles to the service account created:

1. **Storage Admin:** Used for pushing docker images to Google Container
   Registry (GCR).
2. **Cloud Run Admin:** Used for deploying services to Cloud Run.
3. **IAM Service Account user:** Required by Cloud Run to be able to "act as"
   the runtime identity of the Cloud Run application (in this case, our deployer
   service account needs to able to "act as" the GCE default service account).

```sh
gcloud projects add-iam-policy-binding "${PROJECT_ID?}" \
   --member="serviceAccount:${SVCACCT_EMAIL?}" \
   --role="roles/storage.admin"
```

```sh
gcloud projects add-iam-policy-binding "${PROJECT_ID?}" \
   --member="serviceAccount:${SVCACCT_EMAIL?}" \
   --role="roles/run.admin"
```

```sh
gcloud projects add-iam-policy-binding "${PROJECT_ID?}" \
   --member="serviceAccount:${SVCACCT_EMAIL?}" \
   --role="roles/iam.serviceAccountUser"
```

## Step 4: Encrypt the service account key

Run the following command

```sh
travis encrypt-file --pro google-key.json
```

This command will print an `openssl [...]` command, **don’t lose it!**

Edit the `.travis.yml` file, and add this commmand to the `before_install` step:

```diff
 before_install:
-- echo REMOVE_ME # replace with the openssl command from "travis encrypt-file"
+- openssl aes-256-cbc -K $encrypted_fbfaf42b268c_key -iv $encrypted_fbfaf42b268c_iv -in google-key.json.enc -out google-key.json -d
 - curl https://sdk.cloud.google.com | bash > /dev/null
 ...
```

## Step 5: Configure your project ID

Edit the `.travis.yml` and configure the environment variables under the `env:`
key (such as `GCP_PROJECT_ID`, `IMAGE`, and `CLOUD_RUN_SERVICE`).

## Step 6: Commit the changes to your fork

:warning: Do not add `google-key.json` file to your repository as it can be
reached by others.

Make a commit, and push the changes to your fork:

```sh
git add google-key.json.enc .travis.yml
```

```sh
git commit -m "Enable Travis CI"
```

```sh
git push -u origin master
```

## Step 7: View build result

Go to [www.travis-ci.com][tr] and view your build results.

There might be errors that require you to fix.

If the build succeeds, the output of `gcloud run beta deploy` command will show
you the URL your app is deployed on! Visit the URL to see if the application
works!

```
[...]
Deploying container to Cloud Run service [example-app] in project [...] region [us-central1]
Deploying new service...
Setting IAM Policy.....done
Creating Revision......done
Routing traffic........done
Done.
Service [example-app] revision [example-app-00001] has been deployed
and is serving traffic at https://example-app-pwfuv4g72q-uc.a.run.app
```

# Bonusoppgaver

- Se om du du kan få Terraform til å lage alle nødvendige ressurser i GCP for applikasjonen
- Se på Prometheus for Metrics  (https://prometheus.io/) - Vi skal bruke dette når vi forlater "flow" DevOps prinsippene

