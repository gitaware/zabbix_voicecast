<img src="images/logo_no_eu_dark_background.png" alt="VoiceCast" width="280">

# VoiceCast integration for Zabbix

Turn Zabbix notifications into automated phone calls. This webhook media type
sends the notification text to a VoiceCast callflow, which calls the recipient
and reads the message aloud using text-to-speech.

Use it to notify on-call staff about problems and, optionally, call them again
when service is restored. Zabbix controls who receives notifications, their
severity, active periods and escalation rules; VoiceCast handles the call.

VoiceCast is provided by **CloudAware, The Hague, The Netherlands**.
Contact: [voicecast@cloudaware.eu](mailto:voicecast@cloudaware.eu).

## Features

- Importable webhook media type with problem and recovery message templates.
- Spoken notification text, with no email-style subject parameter.
- Existing VoiceCast callflows, voices and languages.
- One international phone number per recipient media entry.
- HTTPS and bearer API-key authentication.
- Asynchronous call queueing: Zabbix does not wait for the conversation to end.
- Optional outbound HTTP proxy support.
- No additional scripts, npm packages or Composer packages to install on the
  Zabbix server for this integration.

## Requirements

| Component | Requirement |
| --- | --- |
| Zabbix | Export targets **Zabbix 7.0**. Validate import and delivery on your installed version; other versions are not claimed as tested. |
| Permissions | Access to configure media types, global macros, users and trigger actions. |
| VoiceCast | An enabled account with an API key and access to a tenant callflow. |
| Call processing | VoiceCast's call dispatcher must be running, with outbound telephony and text-to-speech configured. |
| Connectivity | The Zabbix server must reach the VoiceCast tenant over HTTPS and trust its TLS certificate. |
| Time | Keep the Zabbix and VoiceCast server clocks synchronized. |

The integration uses `POST /api/call/v2` with an explicit UTC `calldate` to queue
the call. Confirm with your VoiceCast administrator that this API and the
dispatcher are available in your installation.

## Installation

### 1. Prepare your VoiceCast callflow

1. Create a VoiceCast account by contacting voicecast@cloudaware.eu.
2. CloudAware will configure the callflow for you (language, retries, webhooks, etc).
3. You will receive all credentials to configure the Zabbix integration.

### 2. Import the media type

1. Download [media_voicecast.yaml](media_voicecast.yaml) from this repository.
   Download the raw file, not the GitHub HTML page.
2. In Zabbix, open **Alerts → Media types → Import**.
3. Select the YAML file and import it.

The imported **VoiceCast** media type starts disabled. Configure it before
enabling it.

### 3. Define the global macros

Open **Administration → Macros** and create:

| Macro | Value |
| --- | --- |
| `{$VOICECAST.URL}` | Your tenant's base URL, such as `https://voicecast.example.com`, without `/api` or surrounding quotes. |
| `{$VOICECAST.API.KEY}` | Your VoiceCast API key. Choose **Secret text** as the macro type. |
| `{$VOICECAST.CALLFLOW}` | The UUID of the callflow prepared in step 1. |

Click **Update** to save. Importing the media type does **not** create these
global macro definitions automatically. Keep the names exactly as shown,
including braces and dollar signs.  
You will receive the value of these macros from CloudAware after you register.

The media type parameters should remain:

| Parameter | Value |
| --- | --- |
| `URL` | `{$VOICECAST.URL}` |
| `APIKey` | `{$VOICECAST.API.KEY}` |
| `Callflow` | `{$VOICECAST.CALLFLOW}` |
| `To` | `{ALERT.SENDTO}` |
| `Message` | `{ALERT.MESSAGE}` |

You can enter literal values for `URL`, `APIKey` and `Callflow` instead of
macros, but global macros keep configuration in one place and let you store
the API key as Secret text.

Enable the **VoiceCast** media type after configuration.

### 4. Add recipients

Under **Users → Users**, edit a recipient and add a media entry:

| Setting | Value |
| --- | --- |
| Type | VoiceCast |
| Send to | One international number, for example `+31612345678`. |
| When active | The times when this person should receive calls. |
| Use if severity | The severities that should result in a call, such as High and Disaster. |
| Enabled | Checked. |

For multiple recipients, configure each user's media and target those users or
their user group in the action. Do not put a comma-separated list in **Send to**.
Recipients need read access to the hosts that generate the notifications.

### 5. Configure a trigger action

1. Open **Alerts → Actions → Trigger actions**.
2. Create or edit an action with conditions matching your hosts and desired
   severities.
3. Add a **Send message** operation targeting your users or user group.
4. Select **VoiceCast** under **Send only to**, and use the media type's default
   message unless you want custom spoken text.
5. Optionally add a **Recovery operation** using VoiceCast to call when the
   problem resolves.
6. Save and enable the action.

The supplied templates cover **trigger problems and recoveries**. Other event
types or update notifications need their own message templates and operations.

## Test your installation

### Test the media type

Open **Alerts → Media types → VoiceCast → Test**. In the test dialog, replace
**all macro placeholders with actual values**:

| Parameter | Test value |
| --- | --- |
| `URL` | Your actual HTTPS tenant base URL. |
| `APIKey` | Your actual API key, not a masked value. |
| `Callflow` | Your actual callflow UUID. |
| `To` | Your own international phone number. |
| `Message` | `This is a VoiceCast test notification.` |

Zabbix's media-type test does not resolve macro placeholders. A defined macro
can still produce `VoiceCast: configure URL (empty or unresolved macro)` here.
Changing values in this dialog affects only the test; keep the macros in the
saved media type for normal action notifications.

**This test queues a real phone call.** A successful response is:

```text
Queued VoiceCast call <uuid>
```

### Test a High alert entirely through the web interface

1. Under **Data collection → Hosts**, choose an enabled test host and create
   an item with these settings:

   | Field | Value |
   | --- | --- |
   | Name | VoiceCast test |
   | Type | Calculated |
   | Key | `voicecast.test` |
   | Type of information | Numeric (unsigned) |
   | Formula | `0` |
   | Update interval | `10s` |

2. Create a trigger on the same host:

   | Field | Value |
   | --- | --- |
   | Name | VoiceCast test alert |
   | Severity | High |
   | Expression | `last(/YOUR_HOST_NAME/voicecast.test)=1` |

   Replace `YOUR_HOST_NAME` with the host's technical **Host name**, not its
   visible name. Keep problem event generation in its default **Single** mode.

3. Confirm that the VoiceCast action matches this host and High severity, and
   that your recipient's media is active and permits High alerts.
4. Wait until **Monitoring → Latest data** shows `0` for the item.
5. Edit its Formula to `1` and save. After configuration refresh and the next
   calculation, check **Monitoring → Problems**, **Reports → Action log**,
   and VoiceCast's call history.
6. Change the Formula back to `0` to resolve the problem. A recovery operation
   can place another call. Wait for recovery before repeating the test.
7. After testing, resolve the problem, then disable the test item and trigger.

This creates real events and runs any matching actions, including other
notification channels and escalation steps.

## Customize the spoken message

Edit the VoiceCast media type's **Message templates**, or provide a custom
message in a Zabbix action. Keep the text short and easy to understand aloud.
For example:

```text
Attention. {EVENT.NAME} on {HOST.NAME}. Severity {EVENT.SEVERITY}.
```

The webhook sends these callflow parameters:

| VoiceCast placeholder | Content |
| --- | --- |
| `{{alert_text}}` | The Zabbix message body, ready for speech. |
| `{{message}}` | The same message body. |
| `{{source}}` | `zabbix`. |

There is no Subject parameter. `{ALERT.SUBJECT}` is not sent or spoken.
The built-in message bodies include `{EVENT.NAME}` to identify the problem.

## Delivery behavior and retries

**Success means queued, not answered.** The API returns after saving the call;
VoiceCast's dispatcher executes it asynchronously. A failed or unanswered call
does not automatically update Zabbix's notification status. Keypad input in a
callflow does not acknowledge a Zabbix event through this integration.

The imported media type uses a **30-second timeout**, **one concurrent webhook
session** and **one attempt**. VoiceCast has no idempotency key for this API:
after a timeout or lost response, the call may already exist. Check VoiceCast's
call history before retrying. Increasing Attempts or configuring repeated
action operations can create additional calls.

## Troubleshooting

| Symptom | What to check |
| --- | --- |
| Empty or unresolved macro | In the Test dialog, enter actual values. For action notifications, check that the global macros are saved and correctly named. |
| HTTP 401 | API key and whether its VoiceCast user is enabled. |
| HTTP 404 | Tenant hostname, base URL and API availability. |
| HTTP 400 | Destination number and whether the callflow UUID belongs to the selected tenant. |
| HTTP 429 or 5xx | Service limits or server errors; check call history before retrying. |
| TLS or network error | Connectivity from the Zabbix server, certificate trust and optional proxy settings. |
| Queued but no phone call | VoiceCast dispatcher, server clocks, scheduled time and call status. |
| Call connects but message is missing | Connected callflow blocks, TTS configuration and `{{alert_text}}` in the TTS Text field. |
| Manual test works but alerts do not | Action conditions, enabled operations, host permissions, recipient active periods and severity filters. |

The webhook does not log the API key or raw API response body. When asking for
help, provide your Zabbix version and the error message, but omit credentials
and recipient information.

## Updating an existing installation

Back up your current media type and record any customized message templates.
Re-import the YAML with updates enabled for the existing media type, then review
its parameters, enabled status, retry settings and templates before testing.
Preserve your configured global macros and any intentional customizations.

## References and contact

- [Zabbix webhook media types](https://www.zabbix.com/documentation/7.0/en/manual/config/notifications/media/webhook)
- [Zabbix calculated items](https://www.zabbix.com/documentation/7.0/en/manual/config/items/itemtypes/calculated)
- [Zabbix media type import/export](https://www.zabbix.com/documentation/7.0/en/manual/xml_export_import/media)
- VoiceCast enquiries: [voicecast@cloudaware.eu](mailto:voicecast@cloudaware.eu)

This is a VoiceCast integration for Zabbix; it does not imply endorsement or
certification by Zabbix.
