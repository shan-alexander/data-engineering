# GCP Intro / Kickstarter

Purpose: To provide an easy-to-follow walkthrough on how to configure a new GCP project for working with BigQuery and Dataform.

## Step 1: Create the GCP project

If you are creating a personal GCP project for experimentation, tinkering, practicing, etc., consider the free trial option: [https://cloud.google.com/free](https://cloud.google.com/free)

Note that no automatic charges will occur, you only start paying if you decide to activate a full, pay-as-you-go account or choose to prepay. We won't enable that during this tutorial.

Once you've signed up with GCP, you'll see "My First Project" in the top left. Click this, and then create a new project. I've named mine "dataform-intro". Switch to this project.

## Step 2: Start a Dataform repo

Navigate to the BigQuery > Dataform tool. Click "Create Repository", give it a hyphenated name (I name the repo "dataform-playground" on my instance). Choose any location for the repo. Upon creating, you'll get a message like this:

![image here]()

Take a screenshot, or copy&paste the name of this service account into a notepad. Mine is `57489593326@gcp-sa-dataform.iam.gserviceaccount.com`. If you fail to write down this ID, you can find the name of the service account in

Note how the message says the service account needs the `roles/bigquery.jobUser` role. It actually needs more than this role, so let's get that process started.

## Step 3: Grant permissions to service account

- roles/iam.serviceAccountTokenCreator

BigQuery Job User: roles/bigquery.jobUser
BigQuery Data Editor: roles/bigquery.dataEditor
BigQuery Data Viewer: roles/bigquery.dataViewer
BigQuery Data Owner: roles/bigquery.dataOwner

Before you click "Save", it should look like:

![image here]()

You can consider other roles like:

Dataform Admin (roles/dataform.admin): For managing Dataform repositories, including creation and deletion.
Composer Worker (roles/composer.worker): If you are using Cloud Composer to schedule Dataform runs.

But we might not use these...

Now go back to Dataform.

## Step 4: Create a workspace

Dataform workspaces can be thought of as branches. They can also be used simply to categorize different efforts, such as creating a separate workspace for data-engineering, data-analysis, and ml-models.

I've seen smaller companies (less than 1000 employees) use workspaces to categorize different data efforts. I've seen huge companies (more than 100k employees) with large data teams (>50 engineers & analysts) use workspaces as personal branches, in which each workspace is an employee's name, and the workspace is used to pull & push syncs with the remote production branch.

For our walkthrough, we'll keep it simple. Name the workspace whatever you like.

Then in your empty workspace, you'll see the option to initialize it.

![image here]()

Click the button to initialize.

![image here]()

You'll now have a basic folder structure.

## Step 5: Return to the Dataform tutorial

Now you're halfway through Step 1 on the [Dataform Turorial Readme](READEME.md). Go there and continue your learning journey!
