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
- For voice alerting, add access permissions in section Calls for: **Make a ring call, Make a TTS call, Make a TTS Advanced call**.
- Save settings.

<br/><br/>
## In Zabbix

The configuration consists of a _media type_ in Zabbix, which will invoke the webhook to send alerts to SMSEagle device through the Rest API.


1\. In the **Administration > Media types** section, import the [media_smseagle.yaml](media_smseagle.yaml).


2\. Open the newly added **SMSEagle** media type and replace all *&lt;PLACEHOLDERS&gt;* with your values.<br>
The following parameters are required:<br>
**access_token** - API access token created in SMSEagle<br>
**url** - actual URL of your SMSEagle device (for example: http://10.10.0.100 or https://sms.mycompany.com)<br>
**type** - type(s) of message(s) to send. Possible values: **sms, mms, tts** and **tts_adv**, respectively for SMS, MMS, TTS Call and Advanced TTS Call.<br/>
Allows multiple types, separated by commas (e.g. "sms,tts_adv").

Other required parameters are message type specific. More information can be found on our [APIv2](https://www.smseagle.eu/docs/apiv2/) page.


3\. in the **Administration > Users** click on a User, and add a new media called **SMSEagle**. Enter SMS recipient. Available recipient formats:<br>
Phone number: <code>phone_number</code><br>
Contact in SMSEagle Phonebook: <code>contact_id:c</code><br>
Group in SMSEagle Phonebook: <code>group_id:g</code><br>

Multiple recipients can be separated by comma.


<br/><br/>
## Voice call tracking (Zabbix 7.0+)

[media_smseagle.yaml](media_smseagle.yaml) only confirms that SMSEagle accepted the message or call into its queue. Zabbix shows the alert as sent even when the call is never completed.

[media_smseagle_call_tracking.yaml](media_smseagle_call_tracking.yaml) (media type **SMSEagle v2 (call tracking)**) also checks the result of TTS calls:

- after queueing a call it polls `GET /api/v2/calls/done` for up to 45 seconds (`verify_timeout_sec`),
- if the call failed (SMSEagle status code 255: not answered, busy or modem error, after retries), the alert is marked as **Failed** and a fallback SMS prefixed with `[VOICE CALL FAILED]` is sent to the same recipients,
- if the call is still in progress after the verification window (e.g. the recipient is listening), the alert is marked as **Sent** and the event gets the tag `smseagle_call_state=pending`,
- call IDs are added as event tags (`smseagle_tts_ids`, `smseagle_tts_adv_ids`), so a Zabbix event can be matched with a call in the SMSEagle web GUI. On a failed alert, Zabbix does not store tags; the IDs are included in the error message in **Reports > Action log**.

Setup differences compared to the standard media type:

- In SMSEagle, the API user additionally needs the **Get calls list** permission (section Calls).
- Import it as a new media type. Set **url** and **access_token**. If SMSEagle runs in a HA cluster, point **url** at the cluster virtual IP.
- Keep **Attempts** at 1. A retry would place a new call and send another fallback SMS.
- Each voice alert occupies a Zabbix alerter for up to about 45 seconds. The media type allows 3 concurrent sessions; keep `StartAlerters` in `zabbix_server.conf` higher than that, otherwise other media types are delayed during alert storms.
- Zabbix limits webhook execution to 60 seconds, so the media type cannot wait for the end of long calls. Failures are normally reported by SMSEagle within seconds, well inside the verification window.
- The message templates are short (severity, host, problem name), because TTS reads the whole message aloud.

Additional parameters:<br>
**verify_calls** - `true` to verify call results (default), `false` to behave like the standard media type<br>
**verify_timeout_sec** - how long to wait for the call result, counted from the start of the script (default 45)<br>
**verify_poll_sec** - polling interval (default 3)<br>
**call_fail_status_codes** - comma separated call status codes treated as failure (default 255)<br>
**fail_on_pending** - `true` to mark the alert as Failed (and send the fallback SMS) when the call has not finished within the window (default false)<br>
**fallback_sms** - `true` to send an SMS when a call fails (default). Not sent when `type` already contains `sms`<br>
**fallback_prefix** - prefix of the fallback SMS text<br>

The parameters `oid`, `date`, `send_after`, `send_before` and `test` of the standard media type are not supported.

<br/><br/>
For more information, please see [Zabbix](https://www.zabbix.com/documentation/6.2/manual/config/notifications) and [SMSEagle](https://www.smseagle.eu/integration-plugins/zabbix-sms-integration/) documentation.
<br/><br/>
## Supported Versions

Zabbix 6.2+ ([media_smseagle.yaml](media_smseagle.yaml))<br>
Zabbix 7.0+ ([media_smseagle_call_tracking.yaml](media_smseagle_call_tracking.yaml))
