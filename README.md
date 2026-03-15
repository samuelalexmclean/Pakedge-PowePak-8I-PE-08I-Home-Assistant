# Pakedge-PowePak-8I-PE-08I-Home-Assistant
Hacky little script to get the Pakedge PowePak 8I PE-08I working in Home Assistant. This has been quite reliable for me and doesn't require HACS, writing your own extension or anything else that looks too much like hard work.

I'm only using the `cgi_getparams`, login and `cgi_commands` endpoints but there's a heap of other interesting stuff going on if you look at developer tools while you're clicking around the web UI. I'm not interested in graphs of power consumption etc but I think it would be easy enough if you felt so inclined.

I'm running `1.03.0.100160`, may work on other models and other firmware revisions.

## Shell interface

Add a new `pdu_control.sh` shell script to your top level HA config directory. I recommend the official Studio Code Server add-on if you're not already using it. You should just need to update your IP and creds. 

```
#!/bin/bash
IP="192.168.0.103"
COOKIE_FILE="/config/.pdu_cookie"
USER="admin" 
PASS="password"

# Log in to get an auth cookie.
do_login() {
    curl -s --compressed -c "$COOKIE_FILE" \
         -d "username=$USER&password=$PASS" \
         "http://$IP/cgi_pdu" > /dev/null
}

# Send a request, and retry it once if auth fails.
send_request() {
    URL=$1
    RESPONSE=$(curl -s --compressed -b "$COOKIE_FILE" "$URL")
    
    # Check if we got JSON. If not, login and retry.
    if [[ ! "$RESPONSE" =~ ^\{ ]]; then
        do_login
        RESPONSE=$(curl -s --compressed -b "$COOKIE_FILE" "$URL")
    fi
    echo "$RESPONSE"
}

# Always respond with the state of each outlet, but optionally toggle first.
if [ "$1" == "toggle" ]; then
    TOGGLE_URL="http://$IP/cgi_commands?group=toggle_outlet&o_id=$2&on_off=$3"
    send_request "$TOGGLE_URL" > /dev/null
    send_request "http://$IP/cgi_getparams?group=show_basic_info"
else
    send_request "http://$IP/cgi_getparams?group=show_basic_info"
fi
```

## Config yaml

Add a switch block to your `configuration.yaml` (or fold this in to existing switch / template blocks, if you have them).

```
switch:
  - platform: template
    switches:
      pdu_outlet_1:
        friendly_name: "Outlet 1"
        value_template: &pdu_state "{{ state_attr('sensor.pdu_data', 'outlets')[0].status == 'on' if state_attr('sensor.pdu_data', 'outlets') else false }}"
        turn_on: &pdu_on
          - action: shell_command.pdu_action
            data: { outlet: 1, state: 1 }
          - action: homeassistant.update_entity
            target: { entity_id: sensor.pdu_data }
        turn_off: &pdu_off
          - action: shell_command.pdu_action
            data: { outlet: 1, state: 0 }
          - action: homeassistant.update_entity
            target: { entity_id: sensor.pdu_data }
```

... and just repeat for all the outlets you want to control. Note that the value in the value_template is 0-indexed and all the others count from 1.

```
      pdu_outlet_2:
        friendly_name: "Outlet 2"
        value_template: "{{ state_attr('sensor.pdu_data', 'outlets')[1].status == 'on' if state_attr('sensor.pdu_data', 'outlets') else false }}"
        turn_on:
          - action: shell_command.pdu_action
            data: { outlet: 2, state: 1 }
          - action: homeassistant.update_entity
            target: { entity_id: sensor.pdu_data }
        turn_off:
          - action: shell_command.pdu_action
            data: { outlet: 2, state: 0 }
          - action: homeassistant.update_entity
            target: { entity_id: sensor.pdu_data }
```

You'll also need a `command_line` and `shell_command` to wire up the bash script:

```
command_line:
  - sensor:
      name: "PDU Data"
      unique_id: pdu_data_master
      command: "/bin/bash /config/pdu_control.sh"
      scan_interval: 43200
      value_template: "{{ value_json.pdu_voltage }}"
      json_attributes:
        - outlets

shell_command:
  pdu_action: "/bin/bash /config/pdu_control.sh toggle {{ outlet }} {{ state }}"
```

Note I'm scanning twice a day, because it will refresh all the states automatically when you toggle a switch via Home Assistant - and that's the only way I toggle switches. You might want a much lower scan interval if you're interfacing with it via other surfaces too.

You can reload your various configs from developer tools, but the quickest way to get this working is just to restart -> look on your home dashboard for the new switches (if you use the automatic one). If not, Developer Tools -> States -> search "PDU" and you should get some helpful info.

Have fun!
