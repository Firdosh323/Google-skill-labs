# Configure Service Accounts and IAM Roles for Google Cloud: Challenge Lab

> Google Skills Boost Challenge Lab covering Service Accounts, IAM Roles, Custom Roles, Compute Engine Service Accounts, and BigQuery access using the Google Cloud CLI.

---

## Lab Information

This walkthrough was completed using the following lab values:

```text
Project ID : qwiklabs-gcp-04-1992635b86ef
Region     : europe-west1
Zone       : europe-west1-d
```

> ⚠️ Google Skills Boost generates unique Project IDs for every lab attempt and may assign different Regions or Zones.
>
> Replace values marked with **CHANGE THIS** using your own lab values.

---

## Your Challenge

In this challenge lab, you will:

- Create Service Accounts using the gcloud CLI
- Grant IAM permissions to Service Accounts
- Create Compute Engine instances with attached Service Accounts
- Create a Custom IAM Role using a YAML file
- Access BigQuery using Service Accounts and Python client libraries

---

# Solving Tasks

## Task 1: Enable and Explore Gemini (Optional)

If Gemini is available in your lab:

1. Open the **Google Cloud Console**
2. Click the **Gemini** icon in the top-right corner
3. Select **Get Gemini Cloud Assist at no cost**
4. Click **Enable Gemini Cloud Assist at no cost**
5. Click **Start Chatting**

Try the following prompts:

```text
What is service account?
```

```text
What is the difference between predefined roles and custom roles?
```

> You can skip this task if Gemini is not required.

---

## Task 2: Create a Service Account Using the gcloud CLI

### SSH into lab-vm

```yaml
gcloud compute ssh lab-vm --zone=europe-west1-d
```

> CHANGE THIS: Replace the zone if your lab uses a different one.

### Activate the Default Configuration

```yaml
gcloud config configurations activate default
```

### Create the Service Account

```yaml
# CHANGE THIS only if your lab specifies another service account name

gcloud iam service-accounts create devops \
--display-name="devops"
```

### Verify

```yaml
gcloud iam service-accounts list
```

### Check Progress

Click **Check my progress**

---

## Task 3: Grant IAM Permissions to a Service Account Using the gcloud CLI

### Store Project ID

```yaml
export PROJECT_ID=$(gcloud config get-value project)
```

### Store Service Account Email

```yaml
export SA=$(gcloud iam service-accounts list \
--filter="displayName=devops" \
--format="value(email)")
```

### Grant Service Account User Role

```yaml
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$SA" \
--role="roles/iam.serviceAccountUser"
```

### Grant Compute Instance Admin Role

```yaml
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$SA" \
--role="roles/compute.instanceAdmin"
```

### Verify

```yaml
gcloud projects get-iam-policy $PROJECT_ID
```

### Check Progress

Click **Check my progress**

---

## Task 4: Create a Compute Instance with a Service Account Attached Using gcloud

### Create VM

```yaml
# CHANGE THIS only if your lab specifies another VM name

gcloud compute instances create vm-2 \
--zone=europe-west1-d \
--service-account=$SA
```

> CHANGE THIS: Replace the zone if your lab uses a different zone.

### SSH into vm-2

```yaml
gcloud compute ssh vm-2 \
--zone=europe-west1-d
```

### Verify Permissions

```yaml
gcloud compute instances list
```

### Check Progress

Click **Check my progress**

---

## Task 5: Create a Custom Role Using a YAML File

### Create role-definition.yaml

```yaml
cat > role-definition.yaml <<EOF
title: Custom Role
description: Custom role with cloudsql.instances.connect and cloudsql.instances.get permissions
includedPermissions:
- cloudsql.instances.connect
- cloudsql.instances.get
EOF
```

### Create the Custom Role

```yaml
# CHANGE THIS only if your lab specifies another custom role name

gcloud iam roles create customRole \
--project=$PROJECT_ID \
--file=role-definition.yaml
```

### Verify

```yaml
gcloud iam roles describe customRole \
--project=$PROJECT_ID
```

### Check Progress

Click **Check my progress**

---

## Task 6: Use the Client Libraries to Access BigQuery From a Service Account

### Create the Service Account

```yaml
# CHANGE THIS only if your lab specifies another service account name

gcloud iam service-accounts create bigquery-qwiklab \
--display-name="bigquery-qwiklab"
```

### Store Service Account Email

```yaml
export BQ_SA=$(gcloud iam service-accounts list \
--filter="displayName=bigquery-qwiklab" \
--format="value(email)")
```

### Grant BigQuery User Role

```yaml
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$BQ_SA" \
--role="roles/bigquery.user"
```

### Grant BigQuery Data Viewer Role

```yaml
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member="serviceAccount:$BQ_SA" \
--role="roles/bigquery.dataViewer"
```

---

### Create the VM Instance

```yaml
# CHANGE THIS only if your lab specifies another VM name

gcloud compute instances create bigquery-instance \
--zone=europe-west1-d \
--service-account=$BQ_SA
```

> CHANGE THIS: Replace the zone if your lab uses a different one.

### SSH into the VM

```yaml
gcloud compute ssh bigquery-instance \
--zone=europe-west1-d
```

---

### Install Dependencies

```yaml
sudo apt-get update

sudo apt-get install -y git python3-pip

pip3 install --upgrade pip

pip3 install google-cloud-bigquery

pip3 install pyarrow

pip3 install pandas

pip3 install db-dtypes
```

---

### Create the Python Script

```yaml
export PROJECT_ID=$(gcloud config get-value project)

cat > query.py <<EOF
from google.auth import compute_engine
from google.cloud import bigquery

credentials = compute_engine.Credentials(
    service_account_email='bigquery-qwiklab@${PROJECT_ID}.iam.gserviceaccount.com')

query = '''
SELECT name, SUM(number) as total_people
FROM `bigquery-public-data.usa_names.usa_1910_2013`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
'''

client = bigquery.Client(
    project='${PROJECT_ID}',
    credentials=credentials)

print(client.query(query).to_dataframe())
EOF
```

> CHANGE THIS only if your lab specifies another state code.

```python
WHERE state = 'TX'
```

### Run the Script

```yaml
python3 query.py
```

Expected output:

```text
      name  total_people
0    James      xxxxxx
1     John      xxxxxx
2   Robert      xxxxxx
...
```

### Check Progress

Click **Check my progress**

---

# Validation Commands

### View Service Accounts

```yaml
gcloud iam service-accounts list
```

### View IAM Policy

```yaml
gcloud projects get-iam-policy $PROJECT_ID
```

### View Custom Roles

```yaml
gcloud iam roles list \
--project=$PROJECT_ID
```

### View VM Instances
