# VoIP Patrol
![GitHub Logo](VP_Logo_1200px-11th_Airborne_Division.patch_small2.jpg)

## VoIP signaling and media test automaton
Designed to automate end2end and or integration tests.

VoIP patrol will follow a scenario in XML format and will output results in JSON.

Each line in the output file is a separate JSON structure, note that the entire file is not a valid JSON file,
this is because VoIP patrol will output results as they become available.

It is possible to test many scenarios that are not easy to test manually like a re-invite with a new codec.

This version is extension of [original project](https://github.com/jchavanton/voip_patrol) and contains changes (in reports and configuration), that are not compatible with original version

### Docker quick start
[quick start with docker](QUICK_START.md)


### Linux Debian building from sources
[see commands in Dockerfile](docker/Dockerfile)

### Load test example
[load test example](load_test/LOAD_TEST.md)

### run
```
./voip_patrol --help
```


### Example: making a test call
```xml
<config>
  <actions>
    <action type="call" label="us-east-va"
            transport="tls"
            expected_cause_code="200"
            caller="15147371787@noreply.com"
            callee="12012665228@target.com"
            to_uri="+12012665228@target.com"
            max_duration="20" hangup="16"
            auth_username="VP_ENV_USERNAME"
            password="VP_ENV_PASSWORD"
            realm="target.com"
            rtp_stats="true"
    >
        <x-header name="X-Foo" value="Bar"/>
    </action>
    <!-- note: param value starting with VP_ENV_ will be replaced by environment variables -->
    <!-- note: rtp_stats will include RTP transmission statistics -->
    <!-- note: x-header tag inside an action will append an header -->
    <action type="wait" complete="true"/>
  </actions>
</config>
```

### Example: starting a TLS server
```bash
./voip_patrol \
   --port 5060 \ # TLS port 5061 +1
   --conf "xml/tls_server.xml" \
   --tls-calist "tls/ca_list.pem" \
   --tls-privkey "tls/key.pem" \
   --tls-cert "tls/certificate.pem" \
   --tls-verify-server \
```
```xml
<config>
  <actions>
     <!-- note: default is the "catch all" account,
          else account as to match called number -->
    <action type="accept"
            match_account="default"
            hangup="5"
            play_dtmf="0123456789#*"
            play="voice_ref_files/f.wav"
            code="200" reason="YES"
            ring_duration="5"
    />
    <!-- DTMF will be sent using RFC2833 -->
    <!-- note: wait for new incoming calls
               forever and generate test results -->
    <action type="wait" ms="-1"/>
  </actions>
</config>
```

### Example: accepting calls and checking for specific header with exact match or regular expression
```xml
<config>
  <actions>
    <action type="accept"
            match_account="default"
            hangup="5"
            code="200" reason="OK"
    >
        <check-header name="Min-SE"/>
        <check-header name="X-Foo" value="Bar"/>
        <check-header name="From" regex="^.*<sip:\+1234@example\.com>"/>
    </action>
    <action type="wait" ms="-1"/>
  </actions>
</config>
```

### Example: accepting calls and searching the message with a regular expression
```xml
<config>
  <actions>
    <action type="accept"
            match_account="default"
            hangup="5"
            code="200" reason="OK"
    >
        <check-message method="INVITE" regex="m=audio(.*)RTP/AVP 0 8.*"/>
        <!-- searching for pcmu pcma in the SDP -->
    </action>
    <action type="wait" ms="-1"/>
  </actions>
</config>
```

### Example: making tests calls with wait_until
Scenario execution is sequential and non-blocking.
We can use “wait” command with previously set “wait_until” params
to control parallel execution.

```
Call States
NULL : Before INVITE is sent or received
CALLING : After INVITE is sent
INCOMING : After INVITE is received.
EARLY : After response with To tag.
CONNECTING : After 2xx is sent/received.
CONFIRMED : After ACK is sent/received.
DISCONNECTED
```
```xml
<config>
  <actions>
    <action type="call" label="call#1"
            transport="udp"
            wait_until="CONFIRMED"
            expected_cause_code="200"
            caller="15148888888@noreply.com"
            callee="12011111111@target.com"
    />
    <!-- note: will wait until all tests pass wait_until state -->
    <action type="wait"/>
    <action type="call" label="call#2"
            transport="udp"
            wait_until="CONFIRMED"
            expected_cause_code="200"
            caller="15147777777@noreply.com"
            callee="12012222222@target.com"
    />
    <action type="wait" complete="true"/>
  </actions>
</config>
```

### Example: testing registration
```xml
<config>
  <actions>
    <!-- note: proxy param to send to a proxy -->
    <action type="register" label="register target.com"
            transport="udp"
            account="VP_ENV_USERNAME"
            username="VP_ENV_USERNAME"
            auth_username="VP_ENV_USERNAME"
            password="VP_ENV_PASSWORD"
            proxy="172.16.7.1"
            realm="target.com"
            registrar="target.com"
            expected_cause_code="200"
    />
    <action type="wait" complete="true"/>
  </actions>
</config>
```

### Example: re-invite with new codec
```xml
<config>
    <action>
        <action type="codec" disable="all"/>
        <action type="codec" enable="pcma" priority="250"/>
        <action type="codec" enable="pcmu" priority="248"/>

        <!-- call that will last 12 seconds and re-invite every 2 seconds -->
        <action type="call"
            wait_until="CONFIRMED"
            expected_cause_code="200"
            caller="16364990640@125.22.198.115"
            callee="12349099229@sip.mydomain.com"
            max_duration="55" hangup="12"
            auth_username="65454659288" password="adaadzWidD7T"
            realm="sip.mydomain.com"
            re_invite_interval="2"
            rtp_stats="true"
        />
        <action type="wait"/> <!-- this will wait until the call is confirmed -->
        <action type="codec" disable="pcma"/>
        <!-- re-invite will now use pcmu forcing a new session -->
        <action type="wait" ms="3000"/> <!-- this will wait 3 seconds -->
        <action type="codec" enable="pcma" priority="250"/>
        <!-- re-invite will now use pcma forcing a new session -->

        <action type="wait" complete="true"> <!-- Wait until the calls are disconnected -->
    <actions/>
<config/>
```

### Example: Overwriting local contact header
```xml
<config><actions>
    <action type="codec" disable="all"/>
    <action type="codec" enable="pcma" priority="250"/>
    <action type="codec" enable="gsm" priority="249"/>
    <action type="codec" enable="pcmu" priority="248"/>

    <action type="call"
        transport="udp"
        caller="+15147371787@fakecustomer.xyz"
        callee="+911@edgeproxy1"
        transport="udp"
        auth_username="20255655"
        password="qntzhpbl"
        realm="sip.flowroute.com"
        rtp_stats="true"
        late_start="false"
        force_contact="sip:+15147371787@10.10.2.5:5777"
        play="/git/voip_patrol/voice_ref_files/reference_8000_12s.wav"
        hangup="5">

    <x-header name="Foo" value="Bar"/>
    </action>
    <action type="wait" complete/>
</actions></config>
```

### Example: WAIT action
#### wait forever:
```xml
<action type="wait" ms="-1"/>
```
#### wait until you receive a certain amount of calls
```xml
<action type="accept" call_count="x" ... />
<action type="wait" complete="true"/>
```
#### wait 5 seconds or one call
```xml
<action type="accept" call_count="1" ... />
<action type="wait" ms="5000"/>
```


### Sample JSON output RTP stats report with multiples sessions
#### one block is generated everytime a session is created
```json
{
 "rtp_stats_0": {
      "rtt": 0,
      "remote_rtp_socket": "10.250.7.88:4028",
      "codec_name": "PCMA",
      "clock_rate": "8000",
      "Tx": {
        "jitter_avg": 0,
        "jitter_max": 0,
        "pkt": 105,
        "kbytes": 16,
        "loss": 0,
        "discard": 0,
        "mos_lq": 4.5
      },
      "Rx": {
        "jitter_avg": 0,
        "jitter_max": 0,
        "pkt": 104,
        "kbytes": 16,
        "loss": 0,
        "discard": 0,
        "mos_lq": 4.5
      }
    },
    "rtp_stats_1": {
      "rtt": 0,
      "remote_rtp_socket": "10.250.7.89:40230",
      "codec_name": "PCMU",
      "clock_rate": "8000",
      "Tx": {
        "jitter_avg": 0,
        "jitter_max": 0,
        "pkt": 501,
        "kbytes": 78,
        "loss": 0,
        "discard": 0,
        "mos_lq": 4.5
      },
      "Rx": {
        "jitter_avg": 0,
        "jitter_max": 0,
        "pkt": 501,
        "kbytes": 78,
        "loss": 0,
        "discard": 0,
        "mos_lq": 4.5
      }
    }
}
```
### Example: email reporting
```xml
<config>
  <actions>
    <action type="alert"
     email="jchavanton+vp@gmail.com"
     email_from="test@voip-patrol.org"
     smtp_host="smtp://gmail-smtp-in.l.google.com:25"
    />
    <!-- add more test actions here ...  -->
    <action type="wait" complete="true"/>
  </actions>
</config>
```

### accept command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| ring_duration | int | ringing duration in seconds |
| early_media | bool | if `true` 183 with SDP and early media is used |
| timer | string | control SIP session timers, possible values are : inactive, optional, required or always |
| code | int | SIP cause code to return must be > `100` and < `700` |
| expected_cause_code | int | SIP cause to be expected from caller side as a call result. Value 487 could be combined with  `fail_on_accept` parameter |
| match_account | string | Account will be used to receive this call (made via `register`) falling back to match the user part of an incoming call RURI or `default` will catch all |
| response_delay | int | delay before 100 reponse is sent in seconds, useful to test timeouts and race conditions |
| call_count | int | The amount of calls to receive to consider the command completed, default `-1` (considered completed) |
| transport | string | Force a specific transport for all messages on accepted calls, default to all transport available |
| play | string | path to file to play upon answer |
| play_dtmf | string | list of DTMF symbols to be sent upon answer |
| re_invite_interval | int | Interval in seconds at which a re-invite with SDP will be sent |
| rtp_stats | bool | if `true` the json report will include a report on RTP transmission |
| min_mos | float | Minimal [MOS](https://en.wikipedia.org/wiki/Mean_opinion_score) value for this call |
| srtp | string | Comma-separated values of the following `sdes` - add SDES support, `dtls` - add DTLS-SRTP support, `force` - make SRTP mandatory |
| cancel | string | `optional` - mark the test passed, if the call was canceled by the caller before answer, `force` - mark test passed ONLY if the call was canceled by the caller. Make sure that you set `ring_duration` > 0 |
| fail_on_accept | bool | If `true` - than accepting this call counts as a failed test |
| hangup | int | call duration in second before hangup |


### call command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| timer | string | control SIP session timers, possible values are : inactive, optional, required or always |
| proxy | string | ip/hostname of a proxy where to send the call |
| caller | string | From header user@host, only used if from it not specified |
| from | string | From header complete `"Display Name" <sip:test at 127.0.0.1>`  |
| callee | string | request URI `user@host` (also used in the To header unless to_uri is specified) |
| to_uri | string | used@host part of the URI in the To header |
| auth_username | string | authentication username on INVITE |
| password | string | password used on INVITE |
| transport | string | force a specific transport `tcp`, `udp`, `tls`, `sips` |
| play | string | path to file to play upon answer |
| play_dtmf | string | list of DTMF symbols to be sent upon answer |
| re_invite_interval | int | Interval in seconds at which a re-invite with SDP will be sent |
| rtp_stats | bool | if `true` the json report will include a report on RTP transmission |
| min_mos | float | Minimal [MOS](https://en.wikipedia.org/wiki/Mean_opinion_score) value for this call |
| srtp | string | Comma-separated values of the following `sdes` - add SDES support, `dtls` - add DTLS-SRTP support, `force` - make SRTP mandatory. Note, if you don't specify `force`, call would be made with plain RTP |
| late_start | bool | if `true` no SDP will be included in the INVITE and will result in a late offer in 200 OK/ACK |
| force_contact | string | local contact header will be overwritten by the given string |
| max_ring_duration | int | max ringing duration in seconds before cancel |
| hangup | int | call duration in second before hangup |
| repeat | int | do this call multiple times |


### register command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| proxy | string | ip/hostname of a proxy where to send the register |
| username | string | AOR username - From/To/Contact header user part |
| auth_username | string | authentication username, account name, From/To/Contact header user part. If not specified, `username` is used |
| password | string | account password |
| account | string | if not specified username is used. Internal identifier, also used in `match_account` in `accept` action |
| registrar | string | SIP UAS handling registration where the messages will be sent |
| transport | string | force a specific transport `tcp`, `udp`, `tls`, `sips` |
| realm | string | realm use for authentication |
| srtp | string | Comma-separated values of the following `sdes` - add SDES support, `dtls` - add "DTLS-SRTP" support, `force` - make SRTP mandatory. Used for incoming calls to this account |
| unregister | bool | unregister the account `<usename@registrar;transport=x>` |
| reg_id | int | if present outbound and other related parameters will be added (see [RFC5626](https://datatracker.ietf.org/doc/html/rfc5626)) |
| instance_id | int | same as `reg_id`, if not present, it will be generated automatically |
| rewrite_contact | bool | default `true`, detect public IP when registering and rewrite the contact header |

### wait command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| complete | bool | if `true` wait for all the test to complete (or reach their wait_until state) before executing next action or disconnecting calls and exiting, needed in most cases |
| ms | int | the amount of milliseconds to wait before executing next action or disconnecting calls and exiting, if `-1` wait forever |

### Example: codec configuration
```xml
<config>
  <actions>
    <action type="codec" disable="all"/>
    <action type="codec" enable="pcmu" priority="250"/>
    <!-- more actions ... -->
    <action type="wait" complete/>
  </actions>
</config>
```

### codec command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| priority | int | 0-255, where zero means to disable the codec |
| enable | string | Codec payload type ID, ex. "g722", "pcma", "opus" or "all" |
| disable | string | Codec payload type ID, ex. "g722", "pcma", "opus" or "all" |

### Example: TURN configuration
```xml
<config>
  <actions>
    <action type="turn" enabled="true" server="x.x.x.x:3478" username="foo" password="bar"/>
    <!-- more actions ... -->
    <action type="wait" complete/>
  </actions>
</config>
```

### turn command parameters

| Name | Type | Description |
| ---- | ---- | ----------- |
| enabled | bool | if "true" turn server usage will be enabled |
| server | string | turn server URI or IP:port |
| username | string | turn server username |
| password | string | turn server password |
| password_hashed | bool | if "true" us hashed password, default plain password |
| sip_stun_use | bool | if "true" SIP reflective IP is use with signaling |
| media_stun_use | bool | if "true" STUN reflective IP is use with media/SDP |
| stun_only | bool | if "true" TURN and ICE are disabled and only STUN is use |

### using env variable in scenario actions parameters
Any value starting with `VP_ENV` will be replaced by the envrironment variable of the same name.
Example : `username="VP_ENV_USERNAME"`
```bash
export VP_ENV_PASSWORD=????????
export VP_ENV_USERNAME=username
```

### Docker
```bash
voip_patrol/docker$ tree
.
├── build.sh        # docker build command example
├── Dockerfile      # docker build file for Linux Alpine
└── voip_patrol.sh  # docker run example starting
```

## Dependencies

#### PJSUA2
PJSUA2 : A C++ High Level Softphone API : built on top of PJSIP and PJMEDIA
http://www.pjsip.org
http://www.pjsip.org/docs/book-latest/PJSUA2Doc.pdf

## External tool to test audio quality

#### PESQ
P.862 : Perceptual evaluation of speech quality (PESQ): An objective method for end-to-end speech quality assessment of narrow-band telephone networks and speech codecs
http://www.itu.int/rec/T-REC-P.862
```
./run_pesq +16000 voice_files/reference.wav voice_files/recording.wav
```
