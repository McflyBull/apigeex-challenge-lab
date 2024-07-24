# Develop and Secure APIs with Apigee X: Challenge Lab (GSP363)

https://partner.cloudskillsboost.google/focuses/32743?parent=catalog

My solution step-by-step and source files.

## Task 1 Solution: Proxy the Cloud Translation API

### Step 1: Setup Using Cloud Shell

In the GCP Console for the Apigee project, open the Cloud Shell and enter the following commands:

```sh
# Enable prerequisite APIs
gcloud services enable compute.googleapis.com
gcloud services enable servicenetworking.googleapis.com
gcloud services enable apigee.googleapis.com

# Enable Cloud Translation API
gcloud services enable translate.googleapis.com

# Enable IAM API
gcloud services enable iam.googleapis.com

# Create service account `apigee-proxy` with role: Logging > Logs Writer
gcloud iam service-accounts create apigee-proxy --display-name="apigee-proxy"
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} --member="serviceAccount:apigee-proxy@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com" --role="roles/logging.logWriter"
echo "Service Account Name for Apigee deployment: apigee-proxy@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com"
```

#### Optional: Running your own Apigee Evaluation Lab

If you are running this in your own Apigee lab (not in Cloud Skills Boost), you'll need to enable an Apigee Evaluation first using the [Apigee UI](https://apigee.google.com). Follow the wizard and assume defaults were appropriate. For the last question in the wizard, do NOT create a Load Balancer (to save costs for testing) - your API will then only be accessible locally, via a test VM.

In the Apigee UI, go to **Admin** > **Environments** > **Groups**, on the `eval-group` click on the *Edit* icon (pencil). In the **Hostnames** field, add a new line: `eval.example.com` and click *Save*. This allows us to replicate the same behaviour as the Challenge Lab by using the same hostname without having to override the Host header or additional parameters.

Create that VM using the following commands in Cloud Shell:

```sh
export PROJECT_ID=$GOOGLE_CLOUD_PROJECT
export AUTH="Authorization: Bearer $(gcloud auth print-access-token)"
export SUBNET=default
export VM_INSTANCE_NAME=apigeex-test-vm
export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
export VM_REGION=$(curl -s -H "$AUTH" https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/instances | jq -r '.instances[0].location')
export VM_ZONE=$(gcloud compute zones list | grep ${VM_REGION} | head -n 1 | awk '{print $2}')

# Create VM
gcloud compute --project=$PROJECT_ID \
    instances create $VM_INSTANCE_NAME \
    --zone=$VM_ZONE \
    --machine-type=e2-micro \
    --subnet=$SUBNET \
    --network-tier=PREMIUM \
    --no-restart-on-failure \
    --maintenance-policy=TERMINATE \
    --preemptible \
    --service-account=$PROJECT_NUMBER-compute@developer.gserviceaccount.com \
    --scopes=https://www.googleapis.com/auth/cloud-platform \
    --tags=http-server,https-server \
    --image=debian-10-buster-v20210122 \
    --image-project=debian-cloud \
    --boot-disk-size=10GB \
    --boot-disk-type=pd-standard \
    --boot-disk-device-name=$VM_INSTANCE_NAME \
    --no-shielded-secure-boot \
    --shielded-vtpm \
    --shielded-integrity-monitoring \
    --reservation-affinity=any


# Connect to VM
gcloud compute ssh $VM_INSTANCE_NAME --zone=$VM_ZONE --project=$PROJECT_ID

### On the VM run the following commands
sudo apt-get update -y
sudo apt-get install -y jq

cat <<EOF >> ~/.bashrc
export AUTH="Authorization: Bearer \$(gcloud auth print-access-token)"
export PROJECT_ID=\$(curl -s -H "Metadata-Flavor: Google" http://metadata.google.internal/computeMetadata/v1/project/project-id)
export ENV_GROUP_HOSTNAME=\$(curl -H "\$AUTH" https://apigee.googleapis.com/v1/organizations/\$PROJECT_ID/envgroups -s | jq -r '.environmentGroups[0].hostnames[0]')
export INTERNAL_LOAD_BALANCER_IP=\$(curl -H "\$AUTH" https://apigee.googleapis.com/v1/organizations/\$PROJECT_ID/instances -s | jq -r '.instances[0].host')
export APIGEE_API_HOST=example.\$PROJECT_ID.apigee.internal
EOF

source ~/.bashrc

sudo -- sh -c "echo $INTERNAL_LOAD_BALANCER_IP $APIGEE_API_HOST eval.example.com >> /etc/hosts"

curl -H "$AUTH" https://apigee.googleapis.com/v1/organizations/$PROJECT_ID | jq -r .caCertificate | base64 -d > cacert.crt
sudo cp cacert.crt /usr/local/share/ca-certificates/apigee-cacert.crt
sudo update-ca-certificates
```

You can test that this works on the VM by issuing the following calls:

```sh
# Send a test request to the hello-world API to verify connectivity
curl -is -H "Host: $ENV_GROUP_HOSTNAME" \
  https://$APIGEE_API_HOST/hello-world \
  --cacert cacert.crt \
  --resolve example.$PROJECT_ID.apigee.internal:443:$INTERNAL_LOAD_BALANCER_IP

# Test using eval.example.com
curl -i -k "https://eval.example.com/hello-world"
```

### Step 2: Apigee UI

- In a new tab, navigate to the Apigee UI: [https://apigee.google.com](https://apigee.google.com)

- Create a *Reverse Proxy* named `translate-v1` with a base path of `/translate/v1` and target URL: `https://translation.googleapis.com/language/translate/v2`. Click *Next*, and then *Next* again - do not make any other changes. Click *Create*.

- Click *Edit Proxy* and then click the *Develop* tab.

- Under *Target Endpoints*, click `default` to edit it. Copy/paste the contents from [targets/default.xml](src/main/apigee/apiproxies/translate-v1/targets/../apiproxy/targets/default.xml) which adds the following section under `HTTPTargetConnection` for Authentication using a GoogleAccessToken:

    ```xml
        <Authentication>
            <GoogleAccessToken>
                <Scopes>
                    <Scope>https://www.googleapis.com/auth/cloud-translation</Scope>
                </Scopes>
            </GoogleAccessToken>
        </Authentication>
    ```

- Click *Save*.

- Click *Deploy to eval* and for Service Account use the one created earlier (outputted in the Cloud Shell)

### Step 3: Test using Cloud Shell

- Run the following in Cloud Shell and **do not proceed** until you see `ORG IS READY TO USE`

    ```sh
    export INSTANCE_NAME=eval-instance; export ENV_NAME=eval; export PREV_INSTANCE_STATE=; echo "waiting for runtime instance ${INSTANCE_NAME} to be active"; while : ; do export INSTANCE_STATE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances/${INSTANCE_NAME}" | jq "select(.state != null) | .state" --raw-output); [[ "${INSTANCE_STATE}" == "${PREV_INSTANCE_STATE}" ]] || (echo; echo "INSTANCE_STATE=${INSTANCE_STATE}"); export PREV_INSTANCE_STATE=${INSTANCE_STATE}; [[ "${INSTANCE_STATE}" != "ACTIVE" ]] || break; echo -n "."; sleep 5; done; echo; echo "instance created, waiting for environment ${ENV_NAME} to be attached to instance"; while : ; do export ATTACHMENT_DONE=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" -X GET "https://apigee.googleapis.com/v1/organizations/${GOOGLE_CLOUD_PROJECT}/instances/${INSTANCE_NAME}/attachments" | jq "select(.attachments != null) | .attachments[] | select(.environment == \"${ENV_NAME}\") | .environment" --join-output); [[ "${ATTACHMENT_DONE}" != "${ENV_NAME}" ]] || break; echo -n "."; sleep 5; done; echo "***ORG IS READY TO USE***";
    ```

- Test the API proxy.
  - In Cloud Shell, open an SSH connection to the `apigeex-test-vm`

    ```sh
    export PROJECT_ID=$GOOGLE_CLOUD_PROJECT
    export AUTH="Authorization: Bearer $(gcloud auth print-access-token)"
    export VM_INSTANCE_NAME=apigeex-test-vm
    export PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
    export VM_REGION=$(curl -s -H "$AUTH" https://apigee.googleapis.com/v1/organizations/$PROJECT_ID/instances | jq -r '.instances[0].location')
    export VM_ZONE=$(gcloud compute zones list | grep ${VM_REGION} | head -n 1 | awk '{print $2}')

    gcloud compute ssh $VM_INSTANCE_NAME --zone=$VM_ZONE --force-key-file-overwrite
    ```

    **NOTE**: Check the value of `$VM_ZONE` - it may not match the lab's expected zone, so you may have to override it if the SSH command above does not connect. As an example, `export VM_ZONE=us-west1-a` and then run the `gcloud compute ssh` command above to connect.

  - On the test VM, run the following command:

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1" \
        -H "Content-Type: application/json" \
        -d '{ "q": "Translate this text!", "target": "es" }'
    ```

  - Observe the response. Note the request and response are being proxied through as-is. It should look similar to the following:

    ```json
    {
        "data": {
            "translations": [
                {
                    "translatedText": "¡Traduce este texto!",
                    "detectedSourceLanguage": "en"
                }
            ]
        }
    }
    ```

### Step 4: Challenge Lab UI

- Click the *Check my progress* button to complete the task.

---

## Task 2 Solution: Change the API request and response

Using the files in this repo, perform the following steps in order using the Apigee UI:

1. Add the Resources using contents from the referenced files:
   1. JavaScript File - [BuildLanguagesResponse.js](src/main/apigee/apiproxies/translate-v1/apiproxy/resources/jsc/BuildLanguagesResponse.js)
   2. Property Set - [language.properties](src/main/apigee/apiproxies/translate-v1/apiproxy/resources/properties/language.properties)
2. Add Policies using contents from the referenced files:
   1. Extension > JavaScript: JS-BuildLanguagesResponse -> Script File: BuildLanguagesResponse.js
   2. Mediation > Assign Message:
      1. [AM-BuildLanguagesRequest](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/AM-BuildLanguagesRequest.xml)
      2. [AM-BuildTranslateRequest](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/AM-BuildTranslateRequest.xml)
      3. [AM-BuildTranslateResponse](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/AM-BuildTranslateResponse.xml)
3. Add Proxy Endpoints - see [proxies/default.xml](src/main/apigee/apiproxies/translate-v1/apiproxy/proxies/default.xml)

Then Save, and Deploy.

Run your tests:

```sh
# List of languages
curl -i -k -X GET "https://eval.example.com/translate/v1/languages"

# Translate to specified language (German)
curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
    -H "Content-Type:application/json" \
    -d '{ "text": "Hello world!" }'

# Translate to default language (Spanish)
curl -i -k -X POST "https://eval.example.com/translate/v1" \
    -H "Content-Type:application/json" \
    -d '{ "text": "Hello world!" }'
```

Back in the Challenge Lab UI:

- Click the *Check my progress* button to complete the task.

**NOTE** In `AM-BuildTranslateResponse`, the response payload JSON needs to be on a single line (no spaces or line breaks) in order for the Challenge's assessment checker to pass. Otherwise, you will be presented with the following error, even though tests using manual API calls yield the correct result:

```text
Please create the 'AM-BuildTranslateResponse' AssignMessage policy with the correct configuration and redeploy the API proxy.
```

---

## Task 3 Solution: Add API key verification and quota enforcement

- Distribution > API Products > +Create. Add API Product `translate-product` - see [products.json](src/tests/translate-v1/products.json)

  - Public access, automatically approve access requests, available in eval environment

  - Operation to allow access to the `translate-v1` proxy using a path of `/` (any request), `GET` and `POST` methods, operation quota of 10 requests per 1 minute

- Distribution > Developers > +Developer. Create a Developer - see [developers.json](src/tests/translate-v1/developers.json)

- Distribution > Apps > +App. Create a Developer App called `translate-app` with the `translate-product` API product associated with the `joe@example.com` developer - see [developerapps.json](src/tests/translate-v1/developerapps.json)
  - Under the Credentials section shown after creation, click *Show* on the *Key* - copy this Key to a safe place (it will be used in testing further down)

- Add Policy: Security > Verify API Key named `VAK-VerifyKey` which should use the `Key` Header. See [VAK-VerifyKey.xml](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/VAK-VerifyKey.xml)

- Add Policy: Traffic Management > Quota named `Q-EnforceQuota`. See [Q-EnforceQuota.xml](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/Q-EnforceQuota.xml)

- On the default Proxy Endpoint, add the PreFlow:

    ```xml
        <PreFlow name="PreFlow">
            <Request>
                <Step>
                    <Name>VAK-VerifyKey</Name>
                </Step>
                <Step>
                    <Name>Q-EnforceQuota</Name>
                </Step>
            </Request>
            <Response/>
        </PreFlow>
    ```

- Save and Deploy to eval.

- Test - Fails (no API key)

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
        -H "Content-Type:application/json" \
        -d '{ "text": "Hello world!" }'
    ```

- Test - Fails (invalid API key)

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
        -H "Content-Type:application/json" \
        -H "Key: ABC123" \
        -d '{ "text": "Hello world!" }'
    ```

- Test - Succeeds (when `$KEY` is set to a valid API Key - get this from the earlier step when setting up a Developer App)

    ```sh
    export KEY=REPLACEWITHVALIDKEY
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
        -H "Content-Type:application/json" \
        -H "Key: $KEY" \
        -d '{ "text": "Hello world!" }'
    ```

- Click the *Check my progress* button to complete the task.

**NOTE** This task currently does NOT pass. A ticket is open with Qwiklabs. The errors returned are:

```text
Please add the 'Q-EnforceQuota' Quota policy with the correct configuration to the proxy endpoint preflow and redeploy the API proxy.
```

---

## Task 4 Solution: Add message logging

- Add Policy: Extension > Message Logging named `ML-LogTranslation` - see [ML-LogTranslation.xml](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/ML-LogTranslation.xml)

- On the default Proxy Endpoint, in the `translate` Flow, after the `AM-BuildTranslateResponse` Step, add the following Step in the Response:

    ```xml
                    <Step>
                        <Name>ML-LogTranslation</Name>
                    </Step>
    ```

- Save and Deploy to eval.

- Test the MessageLogging policy is working:

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
        -H "Content-Type:application/json" \
        -H "Key: $KEY" \
        -d '{ "text": "Hello world!" }'
    ```

- In the GCP Console, visit the Logging page. In the query area, use the *Log name* dropdown to select `translate` and click *Apply*, and then click *Run query* after issuing the API request. There is a short delay, but the log should appear as follows:

    ```text
    de|Hello world!|Hallo Welt!
    ```

- Click the *Check my progress* button to complete the task.

---

## Task 5 Solution: Rewrite a backend error message

- Add Policy: Mediation > AssignMessage named `AM-BuildErrorResponse` - see [AM-BuildErrorResponse.xml](src/main/apigee/apiproxies/translate-v1/apiproxy/policies/AM-BuildErrorResponse.xml)

- On the default Target Endpoint, add a `FaultRules` section after `</HTTPTargetConnection>` as follows:

    ```xml
        <FaultRules>
            <FaultRule name="invalid_request_rule">
                <Step>
                    <Name>AM-BuildErrorResponse</Name>
                </Step>
                <Condition>(fault.name = "ErrorResponseCode")</Condition>
            </FaultRule>
        </FaultRules>
    ```

- Save and Deploy to eval.

- Test - should work

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=de" \
        -H "Content-Type:application/json" \
        -H "Key: $KEY" \
        -d '{ "text": "Hello world!" }'
    ```

- Test - invalid language query parameter, should return the rewritten error message

    ```sh
    curl -i -k -X POST "https://eval.example.com/translate/v1?lang=invalid" \
        -H "Content-Type:application/json" \
        -H "Key: $KEY" \
        -d '{ "text": "Hello world!" }'
    ```

    The error returned should look like this:

    ```json
    { "error": "Invalid request. Verify the lang query parameter." }
    ```

- Click the *Check my progress* button to complete the task.
