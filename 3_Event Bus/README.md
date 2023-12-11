# Module 3: Processing events with EventBridge

## Process events in realtime with EventBridge and Lambda

**Goal:** Implement a rudeimentary flow using EventBridge

<details>
<summary><b>Add EventBridge bus</b></summary><p>

1. Open `serverless.yml`.

2. Add an `EventBridge` bus as a new resource under the `resources.Resources` section

```yml
EventBus:
  Type: AWS::Events::EventBus
  Properties:
    Name: events_tracker_${sls:stage}_${self:custom.name}
```

**IMPORTANT**: make sure that this `EventBus` resource is aligned with `RestaurantsTable` and other CloudFormation resources.

3. While we're here, let's also add the EventBus name as output. Add the following to the `resources.Outputs` section.

```yml
EventBusName:
  Value: !Ref EventBus
```

4. Deploy the project.

`npx sls deploy`

This will provision an EventBridge bus called `order_events_dev_` followed by your name.

</p></details>

<details>
<summary><b>Modify get-restaurants function</b></summary><p>

1. Modify the `get-restaurants` function (in the `functions` section)

```yml
  get-restaurants:
    handler: functions/get-restaurants.handler
    events:
      - http:
          path: /restaurants
          method: get
    environment:
      restaurants_table: !Ref RestaurantsTable
      bus_name: !Ref EventBus
```

Notice that this new function references the newly created `EventBridge` bus, whose name will be passed in via the `bus_name` environment variable.

2. Add the permission to publish events to `EventBridge` by adding the following to the list of permissions under `provider.iam.role.statements`:

```yml
- Effect: Allow
  Action: events:PutEvents
  Resource: !GetAtt EventBus.Arn
```

3. We will need to talk to EventBridge in this new module, so let's install the AWS SDK EventBridge client as a **dev dependency**

```
npm i --save-dev @aws-sdk/client-eventbridge
```

4. Modify `place-order.js` by adding the following in the handler

```javascript
  const { EventBridgeClient, PutEventsCommand } = require('@aws-sdk/client-eventbridge')
  const eventBridge = new EventBridgeClient()

  const busName = process.env.bus_name
  const putEvent = new PutEventsCommand({
    Entries: [{
      Source: 'get-restaurants',
      DetailType: 'tracker',
      Detail: JSON.stringify({
        length: restaurants.length,
        event: 'event'
      }),
      EventBusName: busName
    }]
  })
  console.log(putEvent)
  await eventBridge.send(putEvent)
```



This `get-restaurants` function adds a dummy/trivial event to the event bridge.
</p></details>

<details>
<summary><b>Add process-events function</b></summary><p>

1. Add a new `process-events` function (in the `functions` section)

```yml
  process-events:
    handler: functions/process-events.handler
    environment:
      bus_name: !Ref EventBus
```

Notice that this new function references the newly created `EventBridge` bus, whose name will be passed in via the `bus_name` environment variable.


3. Add a file `process-events.js` to the `functions` folder

4. Modify `process-events.js` to the following

```javascript

module.exports.handler = async (event, context) => {
    console.log('Processing events from EventBridge...')
    console.log(event)
}
```

This `process-events` function handles events on the event bridge and we need to set up trigger rules.

</p></details>


