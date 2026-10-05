# SMSEagle webhook 

This guide describes how to integrate your Zabbix installation with SMSEagle hardware SMS gateway using the Zabbix webhook feature. This guide will provide instructions on setting up a media type, a user and an action in Zabbix.
<br/><br/>
## In SMSEagle

1\. Create a new user in SMSEagle (menu **Users** > **+ Add Users**, user access level: “User”).

2\. Grant API access to the created user:

- Click **Access to API** beside the newly created user.
- Enable **APIv2**
- Generate new token
- For text messages, add access permissions in section Messages for: **Send SMS, Send MMS**.
- For voice alerting, add access permissions in section Calls for: **Make a TTS call, Make a TTS Advanced call** and **Get calls list**. The last one is needed to check the result of each call (see [Voice call tracking](#voice-call-tracking)).
- Save settings.

<br/><br/>
## In Zabbix

The configuration consists of a _media type_ in Zabbix, which will invoke the webhook to send alerts to SMSEagle device through the Rest API.


1\. In the **Alerts > Media types** section (**Administration > Media types** in Zabbix 6.0), import the [media_smseagle.yaml](media_smseagle.yaml).


2\. Open the newly added **SMSEagle v2** media type and replace all *&lt;PLACEHOLDERS&gt;* with your values.<br>
The following parameters are required:<br>
**access_token** - API access token created in SMSEagle<br>
**url** - actual URL of your SMSEagle device (for example: http://10.10.0.100 or https://sms.mycompany.com). If SMSEagle runs in a HA cluster, use the cluster virtual IP.<br>
**type** - type(s) of message(s) to send. Possible values: **sms, mms, tts** and **tts_adv**, respectively for SMS, MMS, TTS Call and Advanced TTS Call.<br/>
Allows multiple types, separated by commas (e.g. "sms,tts_adv").

Optional parameters (leave empty to use SMSEagle defaults):<br>
**priority** - message or call priority<br>
**modem_no** - modem number to use<br>
**date** - scheduled date and time of sending<br>
**send_after**, **send_before** - time window (HH:MM). Applied to messages, and to calls as the call time window<br>
**oid** - message identifier (SMS/MMS)<br>
**encoding** - SMS/MMS encoding: **standard** or **unicode**<br>
**flash** - `true` to send a flash SMS<br>
**test** - `true` to test SMS/MMS sending without sending the message<br>
**duration** - how long the phone rings, in seconds (calls)<br>
**voice_id** - voice model for TTS Advanced calls<br>
**HTTPProxy** - HTTP proxy for requests to SMSEagle<br>

More information can be found on our [APIv2](https://www.smseagle.eu/docs/apiv2/) page.


3\. in the **Users > Users** section (**Administration > Users** in Zabbix 6.0) click on a User, and add a new media called **SMSEagle v2**. Enter SMS recipient. Available recipient formats:<br>
Phone number: <code>phone_number</code><br>
Contact in SMSEagle Phonebook: <code>contact_id:c</code><br>
Group in SMSEagle Phonebook: <code>group_id:g</code><br>

Multiple recipients can be separated by comma.

The default message templates are short (severity, host, problem name, time), because TTS calls read the whole message aloud. Adjust them in the **Message templates** tab of the media type if needed.


<br/><br/>
## Voice call tracking

For TTS and TTS Advanced calls, the media type does not stop when SMSEagle accepts the call into its queue. It checks the result of the call:

- after queueing a call it polls `GET /api/v2/calls/done` for up to 45 seconds (`verify_timeout_sec`),
- if the call failed (SMSEagle status code 255: not answered, busy or modem error, after retries), the alert is marked as **Failed** and a fallback SMS prefixed with `[VOICE CALL FAILED]` is sent to the same recipients,
- if the call is still in progress after the verification window (e.g. the recipient is listening, or the call waits for its time window), the alert is marked as **Sent** and the event gets the tag `smseagle_call_state=pending`,
- call IDs are added as event tags (`smseagle_tts_ids`, `smseagle_tts_adv_ids`), so a Zabbix event can be matched with a call in the SMSEagle web GUI. On a failed alert, Zabbix does not store tags; the IDs are included in the error message in **Reports > Action log**.

Things to know:

- Keep **Attempts** at 1 (tab Options). A retry would place a new call and send another fallback SMS.
- Each voice alert occupies a Zabbix alerter for up to about 45 seconds. The media type allows 3 concurrent sessions; keep `StartAlerters` in `zabbix_server.conf` higher than that, otherwise other media types are delayed during alert storms.
- Zabbix limits webhook execution to 60 seconds, so the media type cannot wait for the end of long calls. Failures are normally reported by SMSEagle within seconds, well inside the verification window.
- SMS and MMS are not tracked: the alert is marked as **Sent** once SMSEagle has queued the message.

Parameters:<br>
**verify_calls** - `true` to check call results (default), `false` to only queue the call<br>
**verify_timeout_sec** - how long to wait for the call result, counted from the start of the script (default 45)<br>
**verify_poll_sec** - polling interval (default 3)<br>
**call_fail_status_codes** - comma separated call status codes treated as failure (default 255)<br>
**fail_on_pending** - `true` to mark the alert as Failed (and send the fallback SMS) when the call has not finished within the window (default false)<br>
**fallback_sms** - `true` to send an SMS when a call fails (default). Not sent when `type` already contains `sms`<br>
**fallback_prefix** - prefix of the fallback SMS text<br>


<br/><br/>
## Upgrading from a previous version

Import the new [media_smseagle.yaml](media_smseagle.yaml) with **Update existing** checked for Media types. The existing **SMSEagle v2** media type is updated in place; user media and actions that use it are kept.

**The import resets all media type parameters to the values from the file**, including **url** and **access_token**, and replaces the message templates. Before importing, note your current parameter values and any changes to the message templates, then enter them again right after the import. Until **url** and **access_token** are set again, alerts fail with `Required parameter is not set`.

Changes in behaviour compared to the previous version:

- TTS calls are tracked (see [Voice call tracking](#voice-call-tracking)). An alert whose call was not completed is now marked as **Failed** and triggers a fallback SMS. Set **verify_calls** to `false` to keep the previous behaviour.
- The API user needs the **Get calls list** permission for call tracking.
- **send_after** / **send_before** are now also applied to calls. Previously they were sent with the wrong field names and ignored for calls.
- Each request contains only the fields its endpoint accepts. Previously all parameters, including **access_token**, were sent in the request body.
- Placeholder values of the previous version (`default`, `YYYYmmDDHHMM`, `HH:MM`) are treated as not set, so parameters copied from the previous version keep working.
- Attempts is set to 1 and concurrent sessions to 3.
- The message templates are shorter and no longer contain the Zabbix event URL.


<br/><br/>
For more information, please see [Zabbix](https://www.zabbix.com/documentation/7.0/manual/config/notifications) and [SMSEagle](https://www.smseagle.eu/integration-plugins/zabbix-sms-integration/) documentation.
<br/><br/>
## Supported Versions

Zabbix 6.0+ (tested on 6.0 and 7.0)
