# Controlling Mitsubishi Electric Air conditioners

### Authentication
These Air-conditioners run through a service called MelView in Australia and New Zealand.
This API works on a cookie system, once authenticated the cookie allows you to perform the same actions you would on the website or app.
You get the cookie by POST to the following address.
https://api.melview.net/api/login.aspx

The body of your request has to contain the following values.
```
{
	"user": "my@email",
	"pass": "mypass",
	"appversion": "3.2.632"
}
```

You should also supply the following header. 
Find your user-agent here http://www.useragentstring.com/

```
{
 "User-Agent":"Mozilla/5.0 (Windows NT 10.0; WOW64; rv:54.0) Gecko/20100101 Firefox/54.0"
}
```

The response should contain a cookie object (in node-red this is under msg.headers["set-cookie"][0])
`auth=XXXX; domain=.melview.net; expires=Mon, 09-Oct-2017 08:26:57 GMT; path=/"`
The auth is the information we need but it is important to know the expiry so we need request a new cookie before that date.

### Rooms
Next we need to get the rooms that the MelView app controls.
We do this by POST to the following URL
https://api.melview.net/api/rooms.aspx

Make sure when you POST to the above address that the header has the cookie in it.
`‘Cookie’: [auth cookie]`

This will return all the current rooms and their current state.
If you have multiple rooms it might be useful to loop through them and store them individually.

Output
``` 
[{
		"buildingid": "9999",
		"building": "Building",
		"bschedule": "0",
		"units": [{
				"room": "ROOM1",
				"unitid": "1234",
				"power": "q",
				"wifi": "3",
				"mode": "1",
				"temp": "19",
				"settemp": "24",
				"status": "",
				"schedule1": 0
			}, {
				"room": "ROOM2",
				"unitid": "456",
				"power": "q",
				"wifi": "3",
				"mode": "3",
				"temp": "18",
				"settemp": "19",
				"status": "",
				"schedule1": 0
			}
		]
	}
]
```
### Unit Capabilities

To find out what the unit is capable of you need to send the command to 
https://api.melview.net/api/unitcapabilities.aspx

This is useful for several reasons. It tells you if it supports functions such as swing.
The minimum and maximum temperatures per preset which can be useful for error handling and localip which is good for 
issuing local commands.

An example to get a room's capabilities would be
```
  {
    "unitid":"123"
  };
```
### Actions

Now we have all the information we need to send and action to a unit.
We use the "unitid" provided from the above response to POST commands to the unit.
As above we need to provide the cookie in the header of the command.

We send the command to https://api.melview.net/api/unitcommand.aspx
The body of the request is the action we want to perform.
We need three things to build the command.
The unitid which we retrieved above. The API version, which is currently 2 and the command we want to send.

Refer to the below table for commands.
An example to turn off a unit would be:
```
  {
    "unitid":"123",
    "v":2,
    "commands":"PW0"
  };
```

If you are operating this over LAN you can also add the parameter ``"lc":1`` this wil send commands immediately to the unit.

It is important to validate your message before sending it out. 
You don't want to set a room to 99 degrees also. Look out for the "status" field inn the room response.
If it shows "COMM" that means there is a communication error and the command won't get processed.

| parameter | type | definition |
|---|---|---|
| mode | *string* |  mode ('MD' 1 - HEAT, 2 - DRY, 3 - Cooling, 7 - FAN, 8 - 'auto') |
| fanSpeed | *string* |  fan speed ('FS' 2 - LOW, 3 - MID, 5 - HIGH, 0 - AUTO)|
| power | *string* | current power mode (PW1 - 'on', PW0 - 'off') |
| zone | *string* | zone power mode (Z[0-8] 1 - 'on', Z[0-8] 0 - 'off') |
| setTemperature | *string* | target temperature 'TS' float |

### Local Commands
If you passed the parameter ``"lc":1`` you would get a lc key in the response this is used to send the command directly to the unit.

Using the localip we got in the unitcapabilites.aspx send an XML message to the local unit:
http://localip/smart

```
<?xml version="1.0" encoding="UTF-8"?>
<CSV>
	<CONNECT>ON</CONNECT>
	<CODE>
		<VALUE>{LC KEY}</VALUE>
	</CODE>
</CSV>
```

This should show instantly on your local unit.

### Closing notes
This is not a definitive list of everything that can be done via MelView.
Different units have different capabilities.

If you notice an error or know of additional functionality feel free to add a PR
