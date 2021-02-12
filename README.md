# Termite Swarm Sensor

![Termite Swarm Sensor](https://i.imgur.com/5i9B7sl.jpg)

Swarming termites is a strange phenomenon that happens during a specific timeframe every year in the southern US city I live in. I'm sure it must happen other places, but I have lived in 4 different states, and this is literally the only place I have ever experienced anything like it. No matter how well you think your house is sealed, if a termite swarm comes by, they will get in - especially if your porch lights (or even inside lights) are on. It's truly borderline old testament stuff. This project creates a sensor that tracks when someone tweets about termites in my city during termite swarm season and sends an actionable alert with the content of the tweet, the option to turn on a "Termite Mode" script that turns off all of the lights, and which gives me the option to reset it or look at more tweets on Twitter. Outside of Home Assistant, I have no programming experience, so I would not be surprised if programming best practices are not in play here. I’m sure there are simpler ways to accomplish this and or streamline.

## Home Assistant Helpers

I used HA’s helpers to set up an input_boolean entity and an input_text entity. 

 I created the input_boolean entity in the UI by going to Configuration/Helpers and clicking the add button, then “Toggle.” I then added a name, in my case “Termite Detector”. This entity will be used to turn on the "Termites May Be Swarming" sensor that would stay “on” until turned off manually or by an automated reset.

I created the input_text entity in the UI by going to Configuration/Helpers and clicking the add button, then “Text.” I then added a name, in my case “Termite Tweets”. This entity will be used to store text (Tweets) delivered by the IFTTT webhook. 

## IFTTT Webhook

This sensor uses an [IFTTT integration](https://www.home-assistant.io/integrations/ifttt/) to trigger a webhook if certain Twitter search terms are met. In testing, there has been an up to 8 minute delay between a tweet and the webhook trigger.

**Disclaimer: when I first set this up, IFTTT was still totally free. Both then and now, these are the only applets I use, so I'm still under the limit on the free account.

### IFTTT Applet Setup

After setting up the IFTTT webhook using the integration instructions linked above, I created two applets in IFTTT. 

##### IF THIS
For the "If this" input, I chose Twitter, linked a twitter account following the IFTTT instructions, and then chose "New tweet from search" for the trigger. I used Twitter's search operators to search for "# "termite" OR "termites" AND "(the name of my city)" -RT". This searches for tweets that contain the words "termite" OR "termites" AND "(the name of my city)" while excluding retweets. 

##### THEN THAT

I then chose the "Webhooks" service for the "then that" action. I filled the "URL" with my webhook address that was created during the IFTTT Home Assistant integration setup, set the "Method" to "POST," "Content Type" to "application/json," and "Body" to 
```
{ 
"action": "call_service", 
"service": "input_boolean.turn_on", 
"entity_id": "input_boolean.termite_detector", 
"text": "tweet", 
"tweet": "@{{UserName}}: {{Text}}" 
}
```
The "action," "service," and "entity_id" data classes are explained in the "USING THE INCOMING DATA" section of the [HA IFTTT integration documentation](https://www.home-assistant.io/integrations/ifttt/). I added two data classes labelled "text" and "tweet" to categorize the webhook for additional automations, and pass the actual triggering tweet and username into Home Assistant, respectively. The webhook as written turns on the "input_boolean.termite_detector" entity without any additional configuration in Home Assistant.

I will create automations later to use the additional incoming data from IFTTT.

# configuration.yaml

There are two sections that have to be added to configuration.yaml for this tracker:

## Template Sensors

First I created a termite_display template sensor which is outputting text based on the state of the input_boolean entity that I created using Helpers. If input_boolean.termite_detector is on, then the sensor will output "Termites May Be Swarming." Otherwise, the sensor will output "Termites Not Detected."

```
# configuration.yaml

##########Termite Alert Sensor##########
- platform: template
  sensors:
    termite_display:
        value_template: >
            {% if is_state('input_boolean.termite_detector', 'on') %}
                Termites May Be Swarming
            {% else %}
                Termites Not Detected
            {% endif %}
```
## Actionable Notifications

This section sets up the part of the project that uses actionable notifications. I had some trouble doing this when I first did it for another project, but it’s fairly well documented here: https://companion.home-assistant.io/docs/notifications/actionable-notifications/

```
# configuration.yaml

ios:
  push:
    categories:
      - name: Termite Alert
        identifier: 'termitealert'
        actions:
          - identifier: 'TURN_ON_TERMITE_MODE'
            title: 'Turn on termite mode'
          - identifier: 'RESET_TERMITE_SENSOR'
            title: 'Reset Termite Sensor'
          - identifier: 'SEE_TERMITE_TWEETS'
            title: 'Check Tweets'
```
# Scripts

I created two scripts for use in this project. One turns off all the lights in the house except a few that are on their lowest brightness. (This is "Termite Mode.") The other resets the termite alert and clears tweets. I made these as scripts so I could control them both through automations and manual input buttons in the UI.

```
termite_mode:
  alias: Termite Mode
  sequence:
	  - service: light.turn_off
	    entity_id: all
	  - service: switch.turn_off
	    entity_id: switch.alley_lights
	  - data:
	      brightness_pct: 01
	    entity_id: light.lamp_left
	    service: light.turn_on
	  - data:
	      brightness_pct: 20
	    entity_id: light.master_bedroom_bedside_lamps
	    service: light.turn_on
reset_termite_alert:
  alias: Reset Termite Mode
  sequence:
	  - service: input_boolean.turn_off
	    entity_id: input_boolean.termite_detector
	  - service: input_text.set_value
	    entity_id: input_text.termite_tweets
	    data:
	      value: " "
   ```
# Automations

I created five automations for the main part of this project, and I’ll do my best to explain them and the logic behind them: (I also have an [Actionable Alexa alert](https://github.com/keatontaylor/alexa-actions/wiki), but I’m not going to cover that in this writeup.)

## Automation Using IFTTT webhook to Send Actionable Notification

When the IFTTT webhook is triggered, this automation checks to make sure that a) it's within 3 hours of sunset or before 10:30 pm, b) it's between April 1st and August 1st, and c) the "input_boolean.termite_detector" entity is “off,” and if all conditions are met, it sends an actionable notification (“Someone is tweeting about termites in (the name of my city). Here is what they said”) to an iOS device with the options to "Turn on termite mode," "Reset Termite Sensor," or "Check Tweets." The automation uses the event_data class I created ("tweet") from the IFTTT webhook to pass the actual tweet through to the push notification. ('{{ trigger.event.data.tweet }}')

![Termite Swarm Actionable Notification](https://i.imgur.com/r4XZg1f.jpg)
```
# automations.yaml

- id: '1589933488026'
  alias: Termite Alert - IFTTT Send Tweet Notification
  description: ''
  trigger:
	  - event_data:
	      text: tweet
	    event_type: ifttt_webhook_received
	    platform: event
  condition:
	  - after: sunset
	    after_offset: -03:00:00
	    condition: sun
	  - condition: template
	    value_template: "{% set n = now() %} {{ n.month >= 4 and n.day >=1\n   or n.month\
	      \ <= 8 and n.day <= 1 }}\n"
	  - before: '22:30:00'
	    condition: time
	  - condition: state
	    entity_id: input_boolean.termite_detector
	    state: 'off'
  action:
	  - data_template:
	      data:
	        push:
	          category: termitealert
	      message: Someone is tweeting about termites in (the name of my city). Here is what they
	        said - '{{ trigger.event.data.tweet }}'
	      title: Potential Termite Swarm Alert
	    service: notify.mobile_app_iphone
```

## Automation Turning on Scripts from Actionable Notification

These two automations are triggered by returns from the actionable notifications "Turn on Termite Mode" (which turns on the "Termite Mode" script and "Check Tweets" (which sends a second notification to the device that will open Twitter to the search terms when clicked). I have these automations in practice going to my phone and my wife's phone, so in order to only send the Twitter link to the phone that requested it, I used the Developer Tools to listen to "ios.notification_action_fired" events to determine the "sourceDevicePermanentID," and used that ID to trigger the notification. (There is a duplicate automation not included in this write up with the ID for my wife's phone.)

```
# automations.yaml

- id: alias: Termite Alert - iOS/Alexa Action Trigger Termite Mode
  description: ''
  trigger:
	  - event_data:
	      actionName: TURN_ON_TERMITE_MODE
	    event_type: ios.notification_action_fired
	    platform: event
	  - event_data:
	      event_id: actionable_notification_termite_mode
	      event_response: ResponseYes
	    event_type: alexa_actionable_notification
	    platform: event
  condition: []
  action:
	  - data: {}
	    service: script.termite_mode
- alias: Termite Alert - Launch Twitter on iPhone
  description: ''
  trigger:
	  - event_data:
	      actionName: SEE_TERMITE_TWEETS
	      sourceDevicePermanentID: ########-####-####-####-############
	    event_type: ios.notification_action_fired
	    platform: event
  condition: []
  action:
	  - data:
	      data:
	        url: https://twitter.com/search?q=%22termite%22%20OR%20%22termites%22%20AND%20%22name%20of%20my%20city%22%20-RT&src=typed_query&f=live
	      message: Click Here.
	      title: See Tweets About Termites in (the name of my city)
	    service: notify.mobile_app_iphone
```

## Automation Resetting Termite Swarm Sensor Each Day or When Reset From Actionable Notification

This automation resets the input_boolean.termite_detector entity to “off” every day at 1:00 a.m., 8:00 am, and 3 hours before sunset, or when it is reset through the actionable notification. It's conditioned to run only if sensor.termite_display is showing "Termites May Be Swarming."

```
# automations.yaml

- alias: Termite Alert - Reset for the day
  description: Resets termite alert at the end of the day if it is triggered
  trigger:
	  - at: 01:00:00
	    platform: time
	  - at: 08:00:00
	    platform: time
	  - event: sunset
	    offset: -03:00:00
	    platform: sun
	  - event_data:
	      actionName: RESET_TERMITE_SENSOR
	    event_type: ios.notification_action_fired
	    platform: event
  condition:
	  - condition: state
	    entity_id: sensor.termite_display
	    state: Termites May Be Swarming
  action:
	  - data: {}
	    entity_id: input_boolean.termite_detector
	    service: script.reset_termite_alert
```

## Automation Setting the Text of the Triggering Tweet

I added this automation to input the text of the tweet that triggered the Termite Swarm Sensor into the input_text.termite_tweets entity that I created in Helpers. This allows me to have a Lovelace card displaying the tweet and when it was sent. This data is pulled from the "tweet" data class that I created in IFTTT.

```
# automations.yaml

- alias: Termite Alert - IFTTT - Input Text Tweet for Sensor
  description: ''
  trigger:
	  - event_data:
	      text: tweet
	    event_type: ifttt_webhook_received
	    platform: event
  condition: []
  action:
	  - data_template:
	      entity_id: input_text.termite_tweets
	      value: '{{ trigger.event.data.tweet }}'
	    service: input_text.set_value
```

# Custom Card for Lovelace UI

I combined everything into a simple custom picture elements card. I had originally used an entity card with three button cards combined into stacks, but found I preferred the look of everything unified together, so I came up with this:

![Termite Swarm Sensor](https://i.imgur.com/5i9B7sl.jpg)

```
cards:
  - elements:
      - entity: sensor.termite_display
        style:
          bottom: 58%
          color: black
          font-size: 26px
          left: 4.5%
          transform: initial
        type: state-label
      - entity: sensor.termite_display
        state_image:
          Termites May Be Swarming: 'https://i.imgur.com/x4O77NF.png'
          Termites Not Detected: 'https://i.imgur.com/QX9GxS8.png'
          unknown: 'https://i.imgur.com/QX9GxS8.png'
        style:
          left: 89%
          top: 22%
          width: 12%
        tap_action:
          action: none
        type: image
      - entity: script.termite_mode
        state_image:
          'off': 'https://i.imgur.com/g6XDKwQ.png'
          'on': 'https://i.imgur.com/g6XDKwQ.png'
        style:
          left: 17%
          top: 74%
          width: 30%
        tap_action:
          action: call-service
          service: script.turn_on
          service_data:
            entity_id: script.termite_mode
        type: image
      - entity: script.termite_mode
        state_image:
          'off': 'https://i.imgur.com/jGz3ryb.png'
          'on': 'https://i.imgur.com/jGz3ryb.png'
        style:
          left: 50%
          top: 74%
          width: 30%
        tap_action:
          action: call-service
          service: script.turn_on
          service_data:
            entity_id: script.reset_termite_alert
        type: image
      - double_tap_action:
          action: url
          url_path: 'https://bit.ly/2zveqUh'
        entity: script.termite_mode
        state_image:
          'off': 'https://i.imgur.com/oEj7vAn.png'
          'on': 'https://i.imgur.com/oEj7vAn.png'
        style:
          left: 83%
          top: 74%
          width: 30%
        tap_action:
          action: url
          url_path: 'https://bit.ly/2SM3Z5r'
        type: image
    image: 'https://i.imgur.com/CNlwZC1.png'
    type: picture-elements
  - content: >-
      ### Most recent tweet ({{
      as_timestamp(state_attr('automation.termite_alert_input_text_tweet_for_sensor',
      'last_triggered'))|timestamp_custom("%-I:%M:%S%p - %B %-d, %Y") }}):

      {{ states.input_text.termite_tweets.state }}
    type: markdown
type: vertical-stack

```
## Final Thoughts:
I've mentioned this project a few times before on some forums, but hadn't decided whether to post it since it's such a specific weird thing. Maybe it can inspire someone to tackle some other problem via Twitter crowdsourcing.

If any of these automations would be helpful to anyone as blueprints, let me know and I'll try to set something up. I did this project in May, so some functionality of Home Assistant has changed since then, but everything still works. I tested it today.
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTY3Mjk4NzUzOV19
-->