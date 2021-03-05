# [Cloud] Providers integration

## Motivation

Allow integrating devices using different providers. Provider can be:

* *IoT protocol gateway* - connecting to the cloud as another device and act on behalf other devices
* *Network server* - exposing it's own integration endpoints for its devices
* *Network service/IoT cloud* - Network servers as a service or IoT cloud provider (The Things Network for example)

Drogue cloud needs to provide:
* A Proper mapping and authorization of devices pushing data through different providers
* A way to route commands to devices from different providers

## Referencing providers

We define a provider as a regular device in our system with its own id and credentials.

```json
{
    "metadata": {
        "application": "myapp",
        "name": "gateway1"
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        }
    }
}
```

When devices are subsequently registered, they can specify one or more providers they are connecting through.

```json
{
    "metadata": {
        "application": "myapp",
        "name": "device1"
    },
    "spec": {
        "providers": ["gateway1", "gateway2"]
    }
}
```

### Labels and selectors

Using Kubernetes style labels and selectors we can group providers and allow them to be easily referenced by devices.


```json
{
    "metadata": {
        "application": "myapp",
        "name": "gateway2",
        "labels": {
            "group": "my-gateways"
        }
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        }
    }
}
```

```json
{
    "metadata": {
        "application": "myapp",
        "name": "device1"
    },
    "spec": {
        "providerSelector": {
            "group": "my-gateways"
        }
    }
}
```

## Data pushing

While pushing data the authorization process goes as follows:

* Provider that uses standard HTTP endpoint will use `as` parameter to define exact device it's representing, like

```bash
echo '{"temp":42}' | http --auth 'gateway1@myapp:hey-rodney' POST https://http.sandbox.drogue.cloud/v1/foo?as=device1
```

* The endpoint calls authentication service with `provider`, `credentials` and `device` trying in a single operation to authenticate the provider and authorize it to represent the device.
* If it's not authenticated and authorized `403 FORBIDDEN` status will be returned.

* In case of the provider that don't use standard HTTP endpoint (like ttn for example) the provider, its credentials and representing device will be parsed from the request and its payload.

## Command Routing

In the opposite direction, we need to provide a way to deliver commands that can connect through different (multiple) providers.

All commands enter Drogue cloud through `Command endpoint`. At the moment (and this will remain the default behavior) the command endpoint transforms the HTTP request into the cloud event and sends it
to the appropriate Knative broker.

The goal is to provide a way for the command endpoint to directly send commands to external providers if that's configured.

### External providers

When commands needs to be *pushed* to external providers, we define `commands->external` information of the provider device `spec` section, like

```json
{
    "metadata": {
        "application": "myapp",
        "name": "ttn-eu-service"
    },
    "spec": {
        "credentials": {
            "credentials": [
                {
                    "pass": "hey-rodney"
                }
            ]
        },
        "commands": {
            "external": [
                {
                "endpoint": "https://integrations.thethingsnetwork.org/ttn-eu/api/v2/down/gateway1/gateway1?key=xxx",
                "headers": {
                    "Authorization": "Basic ...",
                    "Content-Type": "application/json",
                },
                "clientCertificate": "cert",
                "type": "ttn"
            }]
        }
    }
}
```

Now we can define device for the service as

```json
{
    "metadata": {
        "application": "myapp",
        "name": "ttn-device"
    },
    "spec": {
        "providers": ["ttn-eu-service"]
    }
}
```

Command routing process works like this:

* When a *Command endpoint* receives a command it call `lookup` method of the authentication service passing device and its provider(s) (Same as `authenticate` just don't do credential verification)
* Authentication service will check if app, providers and device exist and are not disabled
    * The merged configuration will be returned to the command endpoint including the `commands` section.
    * If there's no `command` info, the command is sent through Knative broker to all internal endpoints.
    * If it contains `command->external` info, it will try to push the command to the external provider and report back success/failure to the original caller.

### Multiple providers

In some cases a device can connect through multiple external providers, like

```json
{
    "metadata": {
        "application": "services",
        "name": "ttn-us-device"
    },
    "spec": {
        "providers": ["ttn-us-east-service", "ttn-us-west-service"],
        "commandPolicy": "all"
    },

}
```

In this case we need resolve to which one we should push a command.

To properly solve this use case we need a *device state* service that will keep track of the last provider/internal endpoint device connected through. This is beyond the scope of this RFC and for now
we can implement a solution that will try to send a command to all specified providers.

## Future

* Design and implement *Device state* service that will be able to provide a last endpoint or external provider device connected through.
* Use *Device state* service to improve command routing (both internally by sending a command only to a specific endpoint or external provider).
* There's a type defined for external provider so we can define proper payload mappers for both data and commands.
