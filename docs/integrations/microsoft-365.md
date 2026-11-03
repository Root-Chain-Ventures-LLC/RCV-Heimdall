# Microsoft 365: cutover meeting invitations

Heimdall can send the pre-cutover and post-cutover meetings to people, and to a whole group, as
Microsoft 365 calendar invitations. This guide is for the Microsoft 365 administrator who sets it up
and for the Heimdall administrator who enters the settings (Settings > Integrations > Microsoft 365).

Heimdall is never required to use it. Until the settings are complete and tested, nothing is sent and
nothing else in Heimdall changes.

Checked against Microsoft Learn on 2026-11-03: the permission names, the event and cancel calls, the
time-zone rules and the Exchange Online cmdlets below. Items marked "not verified" are from
experience and should be confirmed in your tenant (the Test button and `Test-ServicePrincipalAuthorization`
tell you).

## How it works

- Heimdall signs in to Microsoft Entra ID as an application (OAuth 2.0 client credentials), with no
  user present, and calls Microsoft Graph.
- It creates the meeting as an event in **one organizer mailbox's calendar**, for example a NOC
  calendar or a shared mailbox: `POST /users/{organizer}/calendar/events`. Exchange sends the
  invitations from that mailbox.
- Changing a meeting in Heimdall updates that event (`PATCH /users/{organizer}/events/{id}`) and
  Exchange sends the update. Cancelling calls `POST /users/{organizer}/events/{id}/cancel` with a
  comment and Exchange sends the cancellation.
- **A group is invited by its email address**, as one more attendee. Exchange expands it and sends
  the invitation to every member. A distribution list, a mail-enabled security group or a Microsoft 365
  group all work.
- Optionally the event is a Teams meeting (`isOnlineMeeting` with the `teamsForBusiness` provider);
  the join link is kept on the meeting in Heimdall.
- Times are sent to Microsoft in UTC and each invitee sees them in their own time zone. The team's
  planning zone is also written in the invitation text.
- Invitees cannot propose new times (`allowNewTimeProposals` is false).

Heimdall calls exactly these four Graph routes plus the token endpoint. It does not read mail, read
other calendars' events, or touch contacts.

### Why the group is invited and not written to

Writing directly into a Microsoft 365 group's own calendar (`/groups/{id}/calendar/events`) is not
supported with application permissions; Graph supports it with delegated permissions only, which
would need a signed-in user and a refresh token to babysit. Inviting the group's address from the
organizer mailbox gives the same result for the members and needs no user.

## 1. Register the app in Entra ID

1. Microsoft Entra admin center > **Identity > Applications > App registrations > New registration**.
2. Name it, for example `Heimdall meetings`. Supported account types: **Accounts in this
   organizational directory only** (single tenant). No redirect URI: this app never signs a person in.
3. Copy two values from the **Overview** page for Heimdall:
   - **Directory (tenant) ID**, a GUID (a verified domain name such as `contoso.example` also works);
   - **Application (client) ID**, a GUID.

## 2. Grant the permission

1. **API permissions > Add a permission > Microsoft Graph > Application permissions**.
2. Choose **Calendars.ReadWrite** and add it.
3. **Grant admin consent for (your organization)**. The status column must show a green tick. An
   application permission does nothing until an administrator consents.

`Calendars.ReadWrite` is the least privileged permission Microsoft lists for creating and cancelling
events as an application. On its own it reaches **every mailbox in the tenant**; step 5 limits that
to one mailbox, and you should do it.

## 3. Create a client secret

1. **Certificates & secrets > Client secrets > New client secret**. Give it a description such as
   `Heimdall` and choose an expiry.
2. **Copy the secret's Value immediately** (not the Secret ID). Microsoft shows it once.
3. **Put the expiry date in a calendar or your monitoring now.** When the secret expires, every
   Heimdall invitation fails (the Test button reports "Credentials accepted: No") until a new secret is
   created and entered. The new secret has a new value; the old one keeps working until it expires,
   so the swap needs no downtime: create the new secret, enter it in Heimdall, test, then delete the old
   one in Entra ID. Shorter expiries (6 to 12 months) limit what a leaked secret is worth.

## 4. Choose the organizer mailbox

The organizer is the mailbox the invitations come from and where the events live. It must be an
Exchange Online mailbox. A **shared mailbox works** and is the usual choice: it needs no licence, and
the NOC can open it in Outlook to see what Heimdall sent. Avoid a person's own mailbox: the
invitations would come from that person, and the permission scope in step 5 would cover their
calendar.

Enter its primary address in Heimdall as the **Organizer mailbox**.

## 5. Restrict the app to that mailbox

Without this step the app can read and change every calendar in the tenant. Pick one of the two
Exchange Online methods. Run the commands in Exchange Online PowerShell
(`Connect-ExchangeOnline`) as a member of the **Organization Management** role group.

### Preferred: RBAC for Applications

Microsoft's current mechanism; it replaces application access policies.

```powershell
# 1. A pointer in Exchange to the app's service principal. Use the IDs of the ENTERPRISE
#    APPLICATION (Entra ID > Enterprise applications), not the app registration page: the Object ID
#    differs there.
New-ServicePrincipal -AppId <application-client-id> -ObjectId <service-principal-object-id> -DisplayName "Heimdall meetings"

# 2. A scope that contains only the organizer mailbox.
New-ManagementScope -Name "Heimdall organizer" -RecipientRestrictionFilter "PrimarySmtpAddress -eq 'noc@contoso.example'"

# 3. Give the app the calendar role over that scope only.
New-ManagementRoleAssignment -App <application-client-id> -Role "Application Calendars.ReadWrite" -CustomResourceScope "Heimdall organizer"

# 4. Check it. InScope must be True for the organizer and False for anyone else.
Test-ServicePrincipalAuthorization -Identity <application-client-id> -Resource noc@contoso.example
Test-ServicePrincipalAuthorization -Identity <application-client-id> -Resource someone.else@contoso.example
```

Then **remove the tenant-wide consent** from step 2 (Entra ID > the app > API permissions >
Calendars.ReadWrite > Remove permission). Microsoft's rule is that grants from Entra ID and from
Exchange RBAC are added together: leaving the tenant-wide one in place means the scope restricts
nothing. If you keep it because you want an application access policy instead, see the next section.

`PrimarySmtpAddress` as a filter property is not verified in the Microsoft pages checked; the
filterable properties are listed at *Recipient filter properties* in the Exchange documentation. If your
tenant refuses it, use another property that only the organizer mailbox has (a `CustomAttribute1`
value, which Microsoft's own example uses), or a scope on a group (`MemberOfGroup`, which takes the
group's distinguished name and counts direct members only).

### Older: application access policy

Microsoft marks this as replaced and says not to create new ones; it still works and is what many
tenants already have. It restricts only the permissions consented in Entra ID (so keep
`Calendars.ReadWrite` consented in step 2).

```powershell
# A mail-enabled security group that contains only the organizer mailbox.
New-ApplicationAccessPolicy -AccessRight RestrictAccess -AppId <application-client-id> -PolicyScopeGroupId heimdall-organizer@contoso.example -Description "Heimdall meetings: organizer mailbox only"
Test-ApplicationAccessPolicy -Identity noc@contoso.example -AppId <application-client-id>
```

(`Test-ApplicationAccessPolicy` is not verified in the pages checked.) A policy can take a while to take
effect; if the Test button still shows "Organizer mailbox found: Yes" for a mailbox you meant to exclude, wait
and test again.

Whichever method you use, a request for any other mailbox must answer 403. The Test button checks
the organizer only; use `Test-ServicePrincipalAuthorization` or `Test-ApplicationAccessPolicy` for the rest.

## 6. Enter it in Heimdall

Settings > Integrations > **Microsoft 365** (administrators only):

| Field | What to enter |
|---|---|
| Tenant ID or domain | The directory (tenant) ID, or a verified domain such as `contoso.example`. |
| Client ID | The application (client) ID. |
| Client secret | The secret **Value**. Write-only: the page shows "Set" and the last four characters, never the secret. Leave it empty on later saves to keep it; **Remove the stored secret** deletes it. |
| Organizer mailbox | The mailbox from step 4, for example `noc@contoso.example`. |
| Default group address | Optional. The group invited to every meeting unless a meeting names another or none. |
| Add a Teams link to new meetings | On by default. Applies when a meeting is first sent, or updated while it has no Teams link yet; an event that already has a link keeps it. |
| Advanced: cloud hosts | Empty for the global Microsoft cloud. For a national cloud enter the login host and Graph host, for example `login.microsoftonline.us` and `graph.microsoft.us`. |

The secret is sealed with a key derived from `HEIMDALL_SECRET_KEY`, like the other integration
secrets. If that key changes the stored secret can no longer be read and the page says to enter it
again. Changing the tenant, the client ID or the login host asks for the secret again, so a stored
secret is never sent to a new place unseen.

Every save, test and disconnect is written to the audit log (who, what, the result), without the secret.

## 7. The Test button

**Test connection** gets a token and reads the organizer's calendar (`GET /users/{organizer}/calendar`).
It creates nothing and sends nothing. With unsaved edits on the page it tests the values as typed;
otherwise it tests, and remembers, the saved connection. It reports four steps, each Yes, No or Not
checked (the step was never reached), and one sentence on what to fix. Microsoft's own error text is
never shown; only the step and our sentence.

| Step | If it says No |
|---|---|
| **Tenant reachable** | Heimdall could not reach the login host (check the host name, DNS, and that this server may connect out on port 443; behind a proxy, set `HTTPS_PROXY` for the Heimdall process), does not trust its certificate, or Microsoft does not know the tenant (check the tenant ID or domain; for a national cloud check the login host). |
| **Credentials accepted** | Microsoft refused the client ID or secret. Check that you copied the secret **Value** and not the Secret ID, that the secret has not expired, and that the client ID is the application (client) ID of an app in this tenant. |
| **Calendar permission granted** | The app signed in but Graph answered "forbidden". Check that `Calendars.ReadWrite` is an **Application** permission with admin consent (or the RBAC role assignment from step 5 exists), and that the scope or policy includes the organizer mailbox. |
| **Organizer mailbox found** | Graph has no usable mailbox for that address. Check the spelling, that the mailbox exists in Exchange Online (an on-premises mailbox is not reachable this way), and, for a licensed mailbox, that it has a licence. |

"Not checked" on a later step is normal after an earlier failure: fix the first No, then test
again. If Microsoft is throttling, the test says to wait and try again.

## Sending meeting invitations

Once Microsoft 365 is connected and tested, every meeting on a cutover (Meetings on the cutover page) has
an **Invitation** block in its Add and Edit dialog:

- **Send invitation through Microsoft 365** is off for every meeting until someone turns it on, so nothing
  is sent by surprise. The toggle only appears while Microsoft 365 is connected (any operator sees it; only
  administrators see the settings).
- **Group to invite** is the group's address. Empty invites the default group from Settings (shown as the
  placeholder); an address here replaces it for this meeting. **Do not invite a group** invites only the
  people listed under Attendees.
- The invitation goes to the people under Attendees plus the group, from the organizer mailbox, with a Teams
  link when Settings says so. The agenda and a link to the cutover are in the invitation; the notes and
  decisions stay in Heimdall.

What Heimdall does when a meeting changes (the edit is always saved first):

| In Heimdall | In Microsoft 365 |
|---|---|
| Save a meeting with the toggle on | Creates the calendar event and sends the invitation. |
| Change the time, place, attendees, agenda or group | Updates the event; Exchange sends the update. Changing only the notes sends nothing. |
| Status changed to cancelled | Cancels the event, with a comment; Exchange sends the cancellation. |
| Toggle turned off on a meeting already sent | Cancels the event. |
| Status changed back to scheduled, or the toggle turned on again | Sends a new invitation (a new event). |
| Marked held | Nothing: the meeting is over and its event is left alone. |
| Remove the meeting | Cancels the event first. If the cancellation fails the meeting is removed anyway, and the cutover log and the server log say so; cancel it in the organizer's calendar. |

**The state on the meeting.** Under the meeting's details a line shows Microsoft 365's state and time:
*Invitation sent*, *updated*, *cancelled*, *Waiting to try again* or *Invitation not sent*, with the Teams join
link and **Open in Outlook** (the event in the organizer's calendar; only people with access to that mailbox can
open it). When something failed, one plain sentence says what to fix (the same sentences as the Test button's;
Microsoft's own text is never shown).

- **Waiting to try again**: Microsoft was throttling or could not be reached. The meeting is saved. A background
  task tries again after 1, 2, 4 and then every 5 minutes, and stops after 12 failed tries in a row (then it
  reads *Invitation not sent*).
- **Invitation not sent**: something needs fixing (a permission, an expired secret, an address Microsoft
  refused, an event someone deleted in Outlook). Fix it, then press **Retry now** (operators and
  administrators). For a deleted event Retry now sends a new invitation.

Every call to Microsoft that changes something (create, update, cancel) is in the audit trail with who did it,
the meeting, the event id and the outcome, including the failures and the automatic retries; the cutover log
gets a line when an invitation is sent, updated or cancelled, and one when its state turns bad.

**The Heimdall calendar file (.ics) is a separate object.** It keeps its own UID and is not tied to the
Outlook invitation; Heimdall does not switch it to the event's iCalUId. People who received the Outlook
invitation do not need the .ics, and opening both would put the meeting in their calendar twice.


## Threat model

**What the permission lets Heimdall do.** With `Calendars.ReadWrite` consented tenant-wide, the
application (anything holding the client secret, not only Heimdall) can read, create, change and delete
events in **every mailbox's calendar**, including meeting details and attendee lists, and can send
invitations and cancellations that look like they come from any user. That is a real phishing and data
exposure risk, which is why step 5 is not optional in a production tenant. It does not include mail,
files, contacts or Teams messages.

**What Heimdall does with it.** The client calls four fixed routes on one mailbox, the organizer from
Settings; it validates the organizer address and event ids before building a URL and has no way to
reach another mailbox or another route. It sends the meeting's title, agenda, location, time, the
attendee list and a link to the cutover. It reads back only the event id, links and join URL.

**Restricting the app to the organizer mailbox** (step 5) turns a tenant-wide permission into one
calendar. Then a stolen client secret can create, change and cancel events in that one mailbox, which
is a nuisance (a false invitation to the group) and not a breach of anyone's calendar. Use a dedicated
shared mailbox that holds nothing else, so even that exposure is only what Heimdall put there.

**Where the secret lives and how it leaks.** It is typed once, stored AES-256-GCM sealed in
Heimdall's database under a key derived from `HEIMDALL_SECRET_KEY`, and used from Heimdall's server.
The API and the page never return it; the client keeps it and the access token in a type that prints as
`[redacted]` in logs, errors and JSON, and Microsoft's error text is never put into a message, log or
audit entry. Whoever can read Heimdall's database **and** `HEIMDALL_SECRET_KEY` (a compromised Heimdall
host) can recover it. Mitigations: restrict the app as above, use a short secret expiry and rotate
it, watch the service principal's sign-ins in Entra ID, and rotate the secret immediately if Heimdall's host or
backups are exposed. A Conditional Access policy for workload identities that limits sign-in to Heimdall's
egress address helps if your licence includes it (not verified).

**What the invitees see.** Everyone on the invitation, and every member of an invited group, sees the
subject, agenda, location and attendee list. Heimdall's own rule applies to the agenda text: do not put
customer names, addresses or account details in it.

**Network.** Heimdall connects out to the login and Graph hosts over TLS 1.2 or newer, with
certificates verified (there is no "skip verification" option for this integration) and without following
redirects. The unencrypted option exists for lab fakes only and warns on the page.
