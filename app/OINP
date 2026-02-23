# OINP (One Intuit Notification Platform)

> **TL;DR** -- OINP is the central platform that powers all email and in-app notifications in Expert Hiring. You manage templates and notifications through the RFM Portal, and the backend sends them via a simple REST call. This doc covers access, templates, notifications, the backend flow, and how to promote changes to prod.

---

## 1. Getting Access

It is simple -- just ask and you are in.

- Drop a message in the Slack channel **#intuit-iris-oinp**
- Access is **self-served** -- no approvals, no tickets, no waiting
- Once you have access, log in to the [OINP Admin Portal](https://rfmportal-e2e.intuit.com/mint-admin-notification-v2-ui/) and start managing your templates and notifications

---

## 2. OINP Templates

> **Portal:** [https://rfmportal-e2e.intuit.com/mint-admin-notification-v2-ui/](https://rfmportal-e2e.intuit.com/mint-admin-notification-v2-ui/)

Go to **Templates** in the left sidebar to see all templates.

### What You See

![OINP Templates Page](images/oinp-templates.png)

- **Create New Template** button at the top
- **Search bar** to find templates by name
- **Filters** for Service Name, Template Type, and Template Engine
- **Thumbnail / Table** toggle to switch views

Each template card shows:

| Field           | Example                          |
|-----------------|----------------------------------|
| Template Name   | Background Check - Reminder      |
| Service Name    | Expert Hiring 2.0                |
| Template Type   | FRAGMENT or STATIC_SHELL         |
| Template Engine | FreeMarker or Mvel               |

### How Templates Are Configured

Every template has rules that decide **when** it shows up and **who** can use it:

- **Enable Conditions** -- the template only appears when all of these match:
  - Job Family (e.g., CCTS, CTDX, CBDX, CMDX)
  - Country (e.g., USA, CAN)
  - Hire Type (e.g., New Hire, Rehire)
  - Completed Todos (e.g., candidate must be HIRED)
  - Stage (e.g., candidate must be in a specific workflow stage)

- **Permissions** -- which roles can **send** the notification (Recruiter, TAC, CX, Admin, CDI)
- **Edit Permissions** -- which roles can **edit** the email body before sending

---

## 3. OINP Notifications

Go to **Notifications > List Notifications** in the left sidebar.

### What You See

![OINP Notifications Page](images/oinp-notifications.png)

- **Create New Notification** button at the top
- **Filters** for Service, Notification Name, and Event Basis
- **Advanced Search** if you are not sure of the exact name

Each notification card shows the **Notification Name**, a short **Description**, the **Event Basis** it is tied to, a unique **ID**, and the **Date** it was created or last modified. Some cards also display an **Approval Status** (e.g., "PENDING APPROVAL").

### Event Basis at a Glance

| Event Basis                     | What It Is For                           |
|---------------------------------|------------------------------------------|
| ExpertHiringMasterEventData     | Standard workflow emails                 |
| ExpertHiringMasterEventData-QBL | QBL-specific workflow emails             |
| CandidatePortalInvitationEvent  | Candidate portal invitation emails       |
| ExpertHiringCandidatePortal     | Candidate portal notifications           |

---

## 4. Backend Flow

Notifications are sent in two ways -- **manually** by a user, or **automatically** by the system. Both end up at the same place: the OINP Adapter sends a REST call to OINP, and the email goes out.

---

### 4A. Manual Flow (User Clicks "Send")

```mermaid
flowchart TD
    UI["EH Frontend"] -->|"GraphQL Mutation"| Resolver["Mutation Resolver"]
    Resolver --> UseCase["Validate + Fetch Workflow"]
    UseCase --> Helper["Fetch Candidate Data\n(Profile, Job Req, Recruiter, CX, TAC, Offer)"]
    Helper --> Adapter["OINP Adapter"]
    Adapter --> Build["Build Payload\n(Template Metadata + Event Data)"]
    Build --> Send["OINP Service\n(HTTP POST)"]
    Send --> OINP["OINP Platform"]
    OINP --> Inbox["Candidate Inbox"]
    Helper --> History["Save to Workflow History"]
```

**How it works:**

1. User clicks send in the EH UI -- a GraphQL mutation fires with the workflow ID and template ID
2. Backend validates the request and fetches the candidate workflow
3. All required data is gathered -- candidate profile, job requisition, recruiter, CX, TAC, and offer details
4. The OINP Adapter builds the notification payload using template metadata and candidate-specific event data
5. The payload is sent as an HTTP POST to the OINP platform endpoint
6. OINP resolves the template, merges the data, and delivers the email (or in-app notification)
7. A history record is saved so the notification shows up in the workflow's history widget

---

### 4B. Automated Flow (System Sends on Its Own)

These emails fire without any user action. The **Camunda BPMN Workflow Engine** uses timers and condition checks to decide when to send.

```mermaid
flowchart TD
    Camunda["Camunda Workflow Engine"] --> Gateway{"Check Conditions"}
    Gateway -->|"Conditions met"| Timer["Wait for Timer\n(duration or specific date)"]
    Timer -->|"Timer expires"| Task["Fire Notification Task"]
    Task --> Processor["Pick Right Processor\n(by template name)"]
    Processor -->|"Validate business rules"| Helper["Fetch Data + Send"]
    Helper --> Adapter["OINP Adapter"]
    Adapter --> OINP["OINP Platform"]
    OINP --> Inbox["Candidate Inbox"]
    Helper --> History["Save to Workflow History"]
```

**How it works:**

1. Each candidate workflow runs a BPMN process in Camunda with built-in timer events
2. A gateway checks conditions -- e.g., "should we send a welcome email?" or "is CX Keep Warm enabled?"
3. If yes, a timer waits for the configured duration (e.g., 24 hours, 3 days) or a specific date (e.g., 30 days before start date)
4. When the timer fires, the system picks the right notification processor for that template
5. The processor validates business rules (hire type, domain, country) before proceeding
6. From here, the flow is the same as manual -- build payload, send to OINP, save history

#### Common Timer Durations

| Notification               | Wait Time  | What Happens                                |
|----------------------------|------------|---------------------------------------------|
| Welcome Email              | 24 hours   | Sent after hire for New Hire / Rehire       |
| CX Keep Warm               | On date    | Sent 30 days before candidate start date    |
| Phone Screen Reminder      | 3 days     | Reminder if phone screen is not completed   |
| Portal Invitation Reminder | 7 days     | Reminder if candidate has not signed in     |
| BI Request Reminder        | 3 days     | Reminder to initiate background check       |
| Written Offer Reminder     | 3 days     | Reminder to send the written offer          |
| Sign Offer Reminder        | 2 days     | Reminder to sign the written offer          |
| Verbal Offer Reminder      | 3 days     | Reminder to send verbal offer               |
| Connectivity / Speed Test  | 3 days     | Reminder for connectivity test              |
| PTIN Verification          | 3 days     | Reminder for PTIN                           |
| Resume Upload              | 3 days     | Reminder to upload resume                   |
| Sign Addendum              | 3 days     | Reminder to sign addendum                   |

#### Event-Driven Notifications (No Timer)

Some automated emails are triggered instantly by a **workflow state change** instead of a timer:

- **Referral Not Accepted** -- fires when a referred candidate's workflow is closed
- **Referral Verbal Offer Accepted** -- fires when a referred candidate accepts an offer
- **Credential Provided / Not Provided** -- in-app alerts when credential status changes

These run asynchronously on a background thread so they do not block the main workflow.

---

## 5. Prod Promotion

Once your OINP notification or template changes are **tested and working in E2E**, you need to promote them to production. OINP supports a **self-serve promotion** workflow.

### Pre-Requisites

Before you promote, make sure:

- You have done **thorough testing** in E2E and attached screenshots as evidence
- **Throttle configurations** are added to your app ID in OINP
- The notification is in an **approved state** in OINP

### Step-by-Step

```mermaid
flowchart LR
    Test["Test in E2E"] --> Jira["JIRA Ticket Created\n(NOTIF-XXXXX)"]
    Jira --> Evidence["Add Evidence\nScreenshots to JIRA"]
    Evidence --> Portal["Go to Self Promote\nPage on Prod RFM Portal"]
    Portal --> Submit["Enter Jira ID +\nAsset ID and Submit"]
    Submit --> Live["Changes Live in Prod\n(up to 4 hours)"]
```

1. **Test in E2E** -- Make sure your notification works correctly in the E2E environment
2. **JIRA ticket is created** -- A prod promotion JIRA (NOTIF-XXXXX) along with a linked rollback JIRA is created for you

![OINP Promote Page](images/oinp-promote.png)

3. **Add evidence to the JIRA** -- Attach screenshots proving successful E2E testing
4. **Go to the Self Promote page** on the **Prod RFM Portal**:

> **URL:** [https://rfmportal.intuit.com/mint-admin-notification-v2-ui/selfPromote](https://rfmportal.intuit.com/mint-admin-notification-v2-ui/selfPromote)

![OINP Self Promote Page](images/oinp-self-promote.png)

5. **Enter the Jira ID and Asset ID**, then click **Submit**
6. If there are warnings, you will be prompted to confirm before proceeding

### Rollback

Need to undo? Simply promote the **corresponding rollback JIRA** that was automatically linked to your original promotion request.

### Troubleshooting

- If you run into errors during promotion, reach out on **#cc-prod-promotion-help** in Slack
- Changes can take **up to 4 hours** to reflect in prod -- do not panic if it is not instant

### Debugging in Prod (Splunk)

After promotion, you can verify your notification is working using Splunk:

- Use the `ruleName` and `sourceServiceName` filters to track your notification
- Check the [OINP Notification Delivery Dashboard](https://rfmportal.intuit.com) to see delivery metrics and performance

---

## Quick Reference

| Item                     | Value / Location                                                                 |
|--------------------------|----------------------------------------------------------------------------------|
| Access Request           | Slack: **#intuit-iris-oinp** (self-served)                                       |
| OINP Portal (E2E)        | https://rfmportal-e2e.intuit.com/mint-admin-notification-v2-ui/                  |
| OINP Portal (Prod)       | https://rfmportal.intuit.com/mint-admin-notification-v2-ui/                      |
| Self Promote (Prod)      | https://rfmportal.intuit.com/mint-admin-notification-v2-ui/selfPromote           |
| Promotion Help           | Slack: **#cc-prod-promotion-help**                                               |
| Service Name             | Expert Hiring 2.0                                                                |
| Template Types           | FRAGMENT, STATIC_SHELL                                                           |
| Template Engines         | FreeMarker, Mvel                                                                 |
| Key Event Bases          | ExpertHiringMasterEventData, ExpertHiringMasterEventData-QBL, ExpertHiringCandidatePortal |
