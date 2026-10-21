# Quick deploy — Daily SAML Certificate Expiry Digest

[![Azure Logic Apps](https://img.shields.io/badge/Azure_Logic_Apps-Managed_identity-0062AD?logo=microsoft&logoColor=white)](https://learn.microsoft.com/azure/logic-apps/authenticate-with-managed-identity)
[![License: MIT](https://img.shields.io/badge/License-MIT-2EA44F.svg)](../LICENSE)

The fast path to the Logic App described in **[README.md](README.md)**: one ARM template, two commands, instead of clicking through 23 actions by hand.

> [!NOTE]
> Deploying from the template skips the [Designer gotchas](README.md#designer-gotchas) in the main guide entirely — the template writes the workflow definition directly, so the field-serialization bugs documented there never come up here. They only matter if you later open this Logic App in the designer and hand-edit an expression yourself.

> [!IMPORTANT]
> This is a community sample, provided as-is. It isn't a supported Microsoft product. Test it in a lab tenant first. Tenant IDs, subscription IDs, and the tenant domain in every screenshot below have been replaced with placeholders; everything else (steps, errors, layout) is real.

## What you need

Same prerequisites as the main guide — see [What you need](README.md#what-you-need).

## Deploy and verify

### Step 1. Deploy the template

The template [`logic-app/cert-expiry-digest-la.json`](logic-app/cert-expiry-digest-la.json) creates the Logic App with a system-assigned managed identity, the daily trigger at 7:00 AM Central, the Microsoft Graph paging loop, and the Office 365 connection (unauthorized until [Step 6](#step-6-authorize-the-office-365-connection)).

```powershell
az group create --name <resource-group> --location <region>
az deployment group create --resource-group <resource-group> `
  --template-file logic-app/cert-expiry-digest-la.json `
  --parameters digestRecipient=<recipient-address>
```

When the deployment finishes, the resource group lists the Logic App and the connection as **Succeeded**.

### Step 2. Check the Logic App

Open the Logic App's **Overview**. **Status** should read **Enabled**, and **Definition** should read **1 trigger, 23 actions**.

<p align="center"><img src="docs/images/cexp-01-overview.png" alt="Logic App Overview blade showing Enabled status, 1 trigger / 23 actions, and run history" width="800"><br><sub><b>Overview.</b> Essentials shows the resource group, location, and run history. This tenant had already run the deployment twice while a permission grant propagated (see <a href="README.md#troubleshooting">Troubleshooting</a>) — that's why two early runs show Failed.</sub></p>

### Step 3. Confirm the managed identity

Open **Settings &gt; Identity**. **Status** should already read **On** — the template enables it at deployment time — and an **Object (principal) ID** is listed.

<p align="center"><img src="docs/images/cexp-02-identity.png" alt="Identity blade showing System assigned status On and an Object (principal) ID" width="800"><br><sub><b>Settings &gt; Identity.</b> Copy the Object (principal) ID; you'll use it in the next step.</sub></p>

### Step 4. Grant the managed identity `Application.Read.All`

The Logic App calls Microsoft Graph as itself, so there's no client secret to create. Find the identity and the Microsoft Graph app role, then assign the role with admin consent:

```powershell
$mi = az resource show -g <resource-group> -n <logic-app-name> `
  --resource-type Microsoft.Logic/workflows --query identity.principalId -o tsv
$graph = az ad sp show --id 00000003-0000-0000-c000-000000000000 --query id -o tsv
$role = az ad sp show --id 00000003-0000-0000-c000-000000000000 `
  --query "appRoles[?value=='Application.Read.All'].id | [0]" -o tsv
```

Save this as `role-body.json`, with the three values above in place of the placeholders:

```json
{ "principalId": "<mi>", "resourceId": "<graph>", "appRoleId": "<role>" }
```

Then post it:

```powershell
az rest --method post `
  --url "https://graph.microsoft.com/v1.0/servicePrincipals/$mi/appRoleAssignments" `
  --headers "Content-Type=application/json" --body "@role-body.json"
```

> [!NOTE]
> Wait 15–30 minutes before the first run. A run that starts too soon can get `403 Forbidden` / `Authorization_RequestDenied` partway through, because its token was issued before the grant finished propagating. See [Troubleshooting](README.md#troubleshooting) — this happened in the tenant used for these screenshots.

### Step 5. Confirm the permission grant

In the Entra admin center, open **Enterprise applications**, find the Logic App by name (switch **Application type** to include managed identities if the search comes up empty), then open **Permissions**.

<p align="center"><img src="docs/images/cexp-03-permissions.png" alt="Entra admin center Permissions page listing Microsoft Graph Application.Read.All granted via Admin consent" width="800"><br><sub><b>Enterprise applications &gt; this app &gt; Permissions.</b> Microsoft Graph shows <code>Application.Read.All</code> (Read all applications), type Application, granted by Admin consent.</sub></p>

### Step 6. Authorize the Office 365 connection

1. Open **Development Tools &gt; API connections**, and select the Office 365 Outlook connection the template created.
2. Select **Edit API connection**, then **Authorize**. Sign in as the account that sends the digest.
3. Select **Save**. The status should read **Connected**.

Until this step is done, **Send digest** and **Send owner alert** fail, and nothing is sent.

### Step 7. Review the designer

Open the Logic App designer. The top level is five steps: the daily trigger, four **Initialize** variables, the **Until all pages** loop (collapsed — it contains the Graph paging and the per-app, per-certificate checks), and **Send digest**.

<p align="center"><img src="docs/images/cexp-04-designer.png" alt="Logic App designer showing the trigger, four Initialize steps, the collapsed Until all pages loop, and Send digest" width="800"><br><sub><b>Logic app designer</b> (zoomed to fit). The loop's 3 inner actions (Graph paging, per-app certificate check, per-cert alerting) stay collapsed at this level.</sub></p>

### Step 8. Run it and check the history

Select **Run** (or wait for the 7:00 AM trigger). Pay attention to the first run after Step 4 — see the Note above. Then open **Run history**.

<p align="center"><img src="docs/images/cexp-05-run-history.png" alt="Run history list with a selected successful run; every step in the monitoring view shows a green checkmark" width="800"><br><sub><b>Run history.</b> A successful run shows a green check on every step, including the <b>Until all pages</b> loop (it can take several minutes — it pages through every service principal in the tenant) and <b>Send digest</b>.</sub></p>

## Next steps

Deployed and running? Everything else — [sample output](README.md#sample-output), [troubleshooting](README.md#troubleshooting), and [limits](README.md#limits) — is in the main guide, and applies the same way whether you deployed this or built it by hand.

Curious how it works under the hood, or need to recreate it somewhere the template can't run? See [Build it step by step](README.md#build-it-step-by-step) in the main guide.

---

**Author**: Dr. Muataz Awad
