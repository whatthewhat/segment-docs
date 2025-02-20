---
title: Personas SQL Traits
---




SQL Traits allow you to import user or account traits from your data warehouse back into Personas to build audiences, or to enhance Segment data that you send to other Destinations.

SQL Traits are only limited by the data in your warehouse. Because anything you can write a query for can become a SQL Trait, you can add detail to your user and account profiles, resulting in more nuanced personalization.

This unlocks some interesting possibilities to help you meet your business goals.

- To improve your support team's customer satisfaction score (CSAT), you can create a SQL Trait of the most common ticket requests for a customer's industry by joining data from cloud sources like Zendesk and Salesforce.  The resulting SQL Trait helps you anticipate the user's problems and accelerate potential solutions.
- To determine if a user resides in a specific area, you can query address data in your warehouse and send it as a `true` or `false` Trait to a Personas audience.
- To fill gaps in your customer profiles to include information before you implemented Segment, you can import historical Traits from your warehouse.
- To predict a customer's lifetime value (LTV), you can generate a complex query based on demographic and customer data in your warehouse. You can then use that information in a Personas audience to send personalized offers or recommend specific products.
- To inform your outreach efforts, you can use complex queries to build churn or product adoption models.

Check out our [SQL Traits blog post](https://segment.com/blog/sql-traits){:target="_blank"} for more customer case studies.


### Example: Cloud Sources Sync

SQL Traits allow you to import data from [object cloud sources](/docs/connections/sources/#object-cloud-sources) like Salesforce, Stripe, Zendesk, Hubspot, Marketo, Intercom, and more. For example, you can bring in Salesforce Leads or Accounts, Zendesk ticket behavior, or Stripe LTV calculations.

The two examples below show SQL queries you can use to retrieve cloud-source information from your warehouse.

**Salesforce lead import**
If you wanted to import data from the Salesforce leads and contacts table, you can use SQL similar to the following query:

```sql
    select external_id_c as user_id,
    lead_score_c,
    lead_age_c,
    lead_status
    -- …more properties
    from salesforce.leads
```

**Has Open Ticket in Zendesk**

This query computes whether a user has an open ticket.

```sql
    select distinct u.external_id as user_id, true as has_open_ticket
    from zendesk.tickets t
    join zendesk.users u
    on u.id = t.requester_id
    where t.status in ('pending','open','hold','new')
```


## Setting up SQL traits

To use SQL Traits, you need the following:

- a warehouse connected to Segment
- a Personas-enabled Segment workspace
- a user account with access to Personas in that workspace

### Step 1. Set up a warehouse source

Segment supports Redshift, Postgres, Snowflake, Azure SQL, and BigQuery as data warehouse sources for SQL Traits. The setup process for BigQuery is a bit different as it _requires_ a service user.

> info "Safeguard your data"
> For any warehouse, we recommend that you create a separate read-only user for building SQL Traits.

#### Redshift, Postgres, Snowflake, Azure SQL Setup

If you don't already have a data warehouse, follow one of the guides here first:
- [Redshift Getting Started](/docs/connections/storage/catalog/redshift/#getting-started)
- [Postgres Getting Started](/docs/connections/storage/catalog/postgres/#getting-started)
- [Snowflake Getting Started](/docs/connections/storage/catalog/snowflake/#getting-started)
- [Azure SQL Getting Started](/docs/connections/storage/catalog/azuresqldw/#getting-started)

Remember to create a read-only service user!

#### BigQuery Setup

To connect BigQuery to Segment SQL Traits, you must create a service account for Segment to use.

1. Navigate to the Google Developers Console.

2. Click the drop down to the left of the search bar and select the project that you want to connect.

   ![](images/bigquery_setup1.png)

   > **Note**: If you don't see the project you want in the menu, click the account switcher in the upper right corner, and check that you're logged in to the right Google account for the project.

3. Click the menu in the upper left and select **IAM & Admin**, then **Service accounts**.

5. Click **Create service account**.

   ![](images/bigquery_setup2.png)

6. Give the service account a name like `segment-sqltraits`.

7. Under **Project Role**, add _only_ the `BigQuery Data Viewer` and `BigQuery Job User` roles.

   ![](images/bigquery_setup3a.png)

   ![](images/bigquery_setup3b.png)

   > IMPORTANT: Do not add any other roles to the service account. Adding other roles can prevent Segment from connecting to the account.

6. Click **Create Key**.

   ![](images/bigquery_setup4.png)

7. Select `JSON` and click **Create**.

   ![](images/bigquery_setup5.png)

   A file with the key is saved to your computer. Save this, because you'll need it to set up the warehouse source in the next step.

   ![](images/bigquery_setup6.png)

   8. Create new BigQuery Warehouse Source in Personas

   Now you can create a new BigQuery warehouse source, upload the JSON key you just downloaded, and complete the BigQuery setup

### Step 2. Add the warehouse as a Personas Source

Once your warehouse is up and running:

1. Navigate to the Personas settings (Personas > Settings tab > Warehouse Sources), and click **New Warehouse Source**.

   ![](images/warehouse_source_setup1.png)

2. Select the type of warehouse you're connecting.

   ![](images/warehouse_source_setup2A.png)

3. In the next screen, provide the connection credentials, and click **Save**.

  ![](images/warehouse_source_setup3.png)

  If you're connecting a BigQuery warehouse, use the JSON key file that you downloaded as the last step.

## Creating a SQL Trait

Before you create a SQL Trait, you must first preview it to validate your query. If you're new to SQL, try out one of the templates Segment offers.

### Preview the SQL trait

From the Personas screen, go to the Computed Traits tab, and click **New Computed Trait**. Next, choose SQL, and click **Configure**. Select the data warehouse that contains the data you want to query.

If you are sending data from [object cloud sources](/docs/connections/sources/#cloud-apps) to your warehouse, the SQL Traits UI has some pre-made templates you can try out.

![Example template: preview all users with an open Zendesk ticket](images/sql_traits_preview1.png)

<!-- need to actually give a sample here -->

When you're building your query, there are some requirements for the data your query returns.

- The query must return a column with a `user_id`, `email`, or `anonymous_id` (or `group_id` for account traits, if you have Personas for B2B enabled).
- It must return at least one additional trait in addition to `user_id`/`group_id`, and no more than 25 total columns
- The query must not return any `user_id`s with a `null` value, or any duplicate `user_id`s.
- The query must not return more than 10 million rows.
- Each record must be less than 16kb in size to adhere to [Segment's maximum request size](/docs/connections/sources/catalog/libraries/server/http-api/#max-request-size).

A successful preview returns a sample of users and their traits.
If Segment recognizes a user already in Personas, it displays a green checkmark on their profile. You can click that user to view the user's profile. If a user has a question mark, Segment hasn't detected this `user_id` in Personas before.

![Click on a user to check out their profile. If a user has a question mark, we haven't seen this user_id in Personas before](images/sql_traits_preview2.png)


### Configure SQL Trait options

Once you're ready to import the SQL Trait, select the Destinations to which you want to send the data.  If you prefer to build Personas audiences directly from the data instead of sending it to a Destination, click **Skip**.

![Select destinations](images/sql_traits_connect1.png)

Give your SQL Trait a name. This is used as a label for descriptive purposes. If you're importing multiple Traits, give it a name like "Zendesk Traits". The Trait names you use in audience-building or in your downstream tools correspond to the column names from the query.

If you're building Personas audiences from this data, select "Compute without enabled destinations".

Click **Create Computed Trait** to save the Trait.

![](images/sql_traits_connect3.png)
Check **Compute without destinations** if you only want to send to Personas

When you create a SQL Trait, Segment runs the query on the warehouse twice a day by default. (If you're interested in a more frequent or customizable schedule, [contact Segment Support](https://segment.com/help/contact/){:target="_blank"}.)

For each row (user or account) in the query result, Personas sends an identify or group call with all the columns that were returned as Traits. For example, if you write a query that returns `user_id,has_open_ticket, num_tickets_90_days, avg_zendesk_rating_90days` we send an identify call with the following payload:

```sql
    {
      type: 'identify',
      userId: 'u123',
      traits: {
        has_open_ticket: true,
        num_tickets_90_days: 3,
        avg_zendesk_rating_90_days: 8
      }
    }
```

Happy Querying!

## FAQs

### Is there a limit to the result set that can be queried and imported?

The result set is capped at 10 million rows.

### How often does Segment query the customer's data warehouse?

Segment queries the data warehouse every 12 hours by default, but can query up to hourly. [Contact us](https://segment.com/help/contact/) for customized schedules.

### What identifiers can I use to query a list?

You can currently query based on `email`, `user_id` or `anonymous_id`. If Segment doesn't locate a match based on the chosen identifier, it creates a new profile. See more below.

### Can I use SQL Traits to create users in Segment? Or do SQL Traits only append Traits to existing users?

Yes. The Personas engine sends an `identify` call if there is no match between the identifier you chose and an existing record. When this happens, Segment creates a new user profile. (This identify call happens in the back-end, and doesn't show up in your Debugger.)

### Does Personas send identify/group calls on every run?

No. Personas only sends an identify/group call if the values in a row have changed from previous runs.

### I have a large (1M+) query of users to import, should I be worried?

If you're importing a large list of users and traits, you'll need to consider your API call usage as well as volume among the partners receiving your data. These vary depending on our partners, so [reach out to us](https://segment.com/help/contact/) for more information.

### Is there a limit on the size of a SQL Trait's payload?

Yes, Segment limits request sizes to a maximum of 16kb. Records larger than this are discarded.

## Troubleshooting

### I'm getting a permissions error.

You might encounter a similar permissions error:
![](images/troubleshoot1.png)

Segment usually displays this error because you're querying a schema and table that the current user cannot access. To check the table privileges for a specific grantee (user), go to [your warehouse source credentials in Personas](https://app.segment.com/goto-my-workspace/personas/settings/warehouse-sources/) to retrieve the user name. To grant access to a table, an admin usually needs to grant access to both a schema and table through the following similar commands:

```sql
    GRANT USAGE ON SCHEMA ecommerce TO segment_user;
    GRANT SELECT ON TABLE ecommerce.users TO segment_user;
```

Learn more about granting permissions:
- https://www.postgresql.org/docs/9.0/sql-grant.html
- https://stackoverflow.com/questions/17338621/what-grant-usage-on-schema-exactly-do

### I'm seeing a maximum columns error.

![](images/troubleshoot2.png)

Segment currently supports returning only 25 columns. [Contact us](https://segment.com/help/contact/) with a description of your use case if you need to access more.

### I'm seeing a duplicate `user_id` error.

![](images/troubleshoot3.png)

Each query row must correspond to a unique user. Segment displays this error if it detects multiple rows with the same `user_id`. Use a `distinct` or `group by` statement to ensure that each row has a unique user_id.

### I'm seeing some users/accounts in my preview with questions marks. What does that mean?

This could mean one of two things:

**1. Segment doesn't recognize this `user_id`/`group_id` in Personas**

This means for the [sources connected to Personas](https://app.segment.com/goto-my-workspace/personas/settings/sources) Segment has not received any event (identify, track, page etc) with this `user_id`. This could still be a legitimate `user_id` for a number of reasons, but before syncing, make sure you rule out option 2 (below) as sending a different identifier as the `user_id` can corrupt your identity graph.

**2. You have the wrong `user_id` column**

You might be returning a value for `user_id` that is inconsistent with how you  track `user_id` elsewhere. Some customers want to return `email` as the `user_id`, or a partner's tool id as the `user_id`. These conflict with Segment best practices and corrupt the identity graph if you then track `user_id` differently elsewhere in your apps.

If you see only question marks in the preview, and have already tracked data historically with Segment, then you likely have the wrong column. If your cloud source doesn't have the database `user_id`, we recommend using a `JOIN` clause with an internal users table before sending the results back to Segment.
