client-go
=========

A GoLang client SDK for FeatureHub.

Features
--------
* Client interface, with an autogenerated mock so you can unit test with this SDK
* Config type, with validation and defaults
* StreamingClient implementation:
    - Features are made available through "Get" methods:
        - `GetFeature` returns a whole feature (by key) with full metadata (but an untyped value)
        - `GetBoolean` returns a boolean feature (by key), or an error if it is unable to assert the value to a boolean
        - `GetNumber` / `GetRawJSON` / `GetString` as above
    - Levelled Logging (you can choose how verbose to make this)
	- Notifiers can be added for named feature keys, which will trigger a user-provided callback function whenever a feature with this key is updated
	- Notifiers can be added ahead of time (before the client even knows about the feature-keys in question)
* Custom errors (allows you to handle different errors in specific ways)


Usage
-----
Some snippets to get you going.

### Connecting to FeatureHub
There are 3 steps to connecting:
1) Prepare a config
2) Use the config to make a client
3) Tell the client to start handling features

#### Prepare a configuration (with a client):
```go
	// Start by creating a config with your serverAddress and API-key:
	config, err := client.NewConfig(serverAddress, apiKey).WithLogLevel(logrus.WarnLevel).WithWaitForData(true).Connect()
	if err != nil {
		log.Panicf("Error creating config: %s", err)
	}

	// Get a context from this config:
	context := config.NewContext()
```

### Requesting Features
The client SDK offers various `Get` methods to retrieve different types of features:
* `GetBoolean(key)`: returns a true or false
* `GetRawJSON(key)`: returns a serialised JSON object
* `GetNumber(key)`: returns a float64
* `GetString(key)`: returns a string

#### Retrieve a BOOLEAN value:
```go
	someBoolean, err := context.GetBoolean("booleanfeature")
	if err != nil {
		log.Fatalf("Error retrieving a BOOLEAN feature: %s", err)
	}
	log.Printf("Retrieved a BOOLEAN feature: %v", someBoolean)
```

#### Retrieve a JSON value:
```go
	someJSON, err := context.GetRawJSON("jsonfeature")
	if err != nil {
		log.Fatalf("Error retrieving a JSON feature: %s", err)
	}
	log.Printf("Retrieved a JSON feature: %s", someJSON)
```

#### Retrieve a NUMBER value:
```go
	someNumber, err := context.GetNumber("numberfeature")
	if err != nil {
		log.Fatalf("Error retrieving a NUMBER feature: %s", err)
	}
	log.Printf("Retrieved a NUMBER feature: %f", someNumber)
```

#### Retrieve a STRING value:
```go
	someString, err := context.GetString("stringfeature")
	if err != nil {
		log.Fatalf("Error retrieving a STRING feature: %s", err)
	}
    log.Printf("Retrieved a STRING feature: %s", someString)
```


### Configuring Notifiers (callbacks)
The client SDK allows the user to define callback notifications which will be triggered whenever a specific feature key is updated.
Notifiers can be defined at any time, even before the client has received data.
* `AddNotifierBoolean(key string, callback func(bool))`: Calls the provided function with a boolean value
* `AddNotifierFeature(key string, callback func(*models.FeatureState))`: Calls the provided function with a raw feature state
* `AddNotifierJSON(key string, callback func(string))`: Calls the provided function with a JSON string value
* `AddNotifierNumber(key string, callback func(float64))`: Calls the provided function with a float64 value
* `AddNotifierString(key string, callback func(string))`: Calls the provided function with a string value
* `DeleteNotifier(key string) error`: Deletes any configured notifier for the given key (or returns an error if no notifier was found)


### Configuring a Readiness Listener
The client SDK allows the user to define a callback function which will be triggered once, when the client first receives some data from the server.
* `ReadinessListener(callback func())`: Sets the readiness listener to a specific user-provided function


### Analytics Collector
The client SDK provides the ability to generate analytics events with the `LogAnalyticsEvent` method. An event will be generated for each feature that we have.

```go
	action := "payment"
	tags := map[string]string{"user": "bob"}
	client.LogAnalyticsEvent(action, tags)
```
The SDK offers a logging analytics collector which will log events to the console at DEBUG level (useful in your unit tests probably).


#### Google Analytics
The GoLang SDK comes with a pre-made Google Analytics collector. Here is how to use it:

```go
	googleAnalyticsCollector, err := analytics.NewGoogleAnalyticsCollector(clientID, trackingID, userAgentKey)
	if err != nil {
		panic(err)
	}
	client.AddAnalyticsCollector(googleAnalyticsCollector)
```
Any subsequent calls to `client.LogAnalyticsEvent()` will result in events being sent via the Google Analytics collector (as well as any other which you have added).


### Client-side rollout strategies
Some rollout strategies need to be calculated per-request, which means that we can't rely on the server to do this for us. For this we provide the ability to apply a client context to a feature before using its value:

```go
	// Add some values to the context:
	context.Country = "russia"
	context.Custom["test"] = true

	// Now retrieved feature values will be evaluated against your context:
	featureValue, err = context.GetString("featureKey")
```

If you have a complex context then you can define it as a single struct:

```go
	// You can also define a context as a struct:
	clientContext := &Context{
		Userkey:  "12345",
		Device:   "server",
		Platform: "linux",
		Country:  "New Zealand",
		Version:  "1.0.5",
		Custom: map[string]interface{}{
			"startDate": "now",
			"username":  "prawn",
			"iteration", float64(5),
		},
	}

	// And get the config to return you a fully-loaded context:
	context := config.WithContext(clientContext)

	// Now retrieved feature values will be evaluated against your context:
	featureValue, err = context.GetString("featureKey")
```

If the featureValue has rollout strategies defined then they will be applied according to the client context you provide.

Note the map of `Custom` values, which are evaluated against your custom features according to their field names (keys).
