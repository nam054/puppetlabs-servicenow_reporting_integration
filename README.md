# puppetlabs-servicenow_reporting_integration

The ServiceNow reporting integration module ships with a `servicenow` report processor that can send one of two kinds of information back to Servicenow. It can send events that are handled by Servicenow to create Alerts and Incidents, or you can create Incidents directly.

#### Table of Contents

1. [Pre-reqs](#pre-reqs)
2. [Setup](#setup)
    * [Events](#events)
    * [Incidents](#incidents)
3. [Troubleshooting](#troubleshooting)
4. [Development](#development)
5. [Known Issues](#known-issues)

## Pre-reqs

* Puppet Enterprise version 2019.8.1 (LTS) or newer
* A ServiceNow instance (dev or enterprise)

## Setup

1. Install the `puppetlabs-servicenow_reporting_integration` module on your Puppet server.

The module can send events or it can create incidents, but attempting to do both will cause a catalog compilation failure. Please choose which one of the two you would like to do and only classify your Puppet server nodes with one of those classes, either event management, or incident management.

### Events

To send events, classify your Puppet servers with the `servicenow_reporting_integration::event_management` class. The minimum parameters you need to configure for the class to work are `instance` (the fqdn of the Servicenow instance), and then `user` and `password`, or you can bypass username/password authentication and use an oauth token using the `oauth_token` parameter.

By default each event will include the following information:
* __Source__: Puppet
* __Node__: The node the agent ran on
* __Type__: The type of event. Can be one of `node_report_intentional_changes`, `node_report_corrective_changes`, `node_report_unchanged`, `node_report_failed`
* __Source instance__: The name of the Puppet server that generated the report
* __Message Key__: A hash of all of the relevant report properties to ensure that future events are grouped together properly
* __Severity__: The highest severity rating of all of the events that occurred in a given run. These severity levels can be configured via the `<change_type>_event_severity` class parameters, including the severity of `no_changes` reports via the `no_changes_events_severity` parameter.
* __Description__: Contains the following:
  * __Report Labels__ - All of the different kinds of events that were in the Puppet run that generated this event. While severity is only the single highest severity, these labels tell you all of the kinds of events that occurred.
  * __Environment__ - The Puppet environment the node is assigned to.
  * __Resource Statuses__ - The full name of the resource, the name of the property and the event message, and the file and line where the resource was defined, for each resource event that was in the report. All resource events are included except for `audit` events, which are resources for which nothing interesting happened; Puppet simply verified that they are currently still in the correct state.
  * __Facts__ - All of the facts that were requested via the `include_facts` parameter. The default format is yaml, but can be changed via the `facts_format` parameter.
* __Additional Information__: The additional information field contains data about the event in JSON format to make it easy to target that information for rules and workflows. It will contain separate top level keys for each of the facts included in the event from the `include_facts` parameter. It will also contain the following keys
  * __environment__: The environment the node is assigned to
  * __report_labels__: A list of all of the different kinds of events that are included in this event.
  * __corrective_changes__: All of the resource events that were triggered by corrective changes.
  * __intentional_changes__: All of the resource events that were triggered by intentional changes.
  * __pending_corrective_changes__: All of the resource events that were trigged by pending (noop) corrective changes.
  * __pending_intentional_changes__: All of the resource events that were triggered by pending (noop) intentional changes.
  * __failures__: All of the resources events that were triggered by resource failures.

The module will send a single event for every Puppet run on every node. If nothing interesting such as changes or a failure happened in a given Puppet run then the event type will be `node_report_unchanged`, there will be no resources listed in the description, and the report severity default value will be `OK`.

If a change happens then the event type and severity will be updated, and any resources that changed will be listed in the description.

If multiple events happen e.g. two resources make corrective changes, but a third resource fails, then the report type will be `node_report_failure`, the severity by default will be `Minor`. All three resources, the resource message that describes what happened, and the file and line where the resource can be found, will be included in the event description.

Event severities can be configured via the `<change_type>_event_severity` class parameters.

You can specify the set of facts included in the event description via the `include_facts` parameter. It takes an array of strings that each represent a fact to retrieve from the available node's facts. Nested facts can be queried by dot notation such as `os.distro`, or `os.windows` to get the Windows version. Queries for nested facts must start at the top level fact name and any fact that is not present on a node, such as `os.windows` on a Linux box, is simply ignored without error.

Facts in the description by default are in Yaml format for readability, but this can be changed via the `facts_format` parameter to one of yaml, pretty_json (json with readability line breaks and indentation), or json.

To stop the module from sending any events to Servicenow, you can set the `disabled` parameter to `true`. Simply removing the module from classification on a Puppet server node will not stop the report processor from continuing to send events to Servicenow. To remove the integration permanently, uninstall the module from the environment and remove `servicenow` from the `reports` setting in `puppet.conf`.

### Incidents

To send incidents, classify your Puppet server nodes with the `servicenow_reporting_integration::incident_mangement` class. The minimum parameters you need to configure for the class to work are `instance` (the fqdn of the Servicenow instance), and then `user` and `password`, or you can bypass username/password authentication and use an oauth token using the `oauth_token` parameter. Lastly you will need to get the `sys_id` of the user you would like to use as the 'Caller' for each ticket. The `servicenow_reporting_integration::incident_management` class requires the `sys_id` for a user to be placed in the the `caller_id` parameter because that is a required incident field on ServiceNow’s end. Look below for steps to get the `sys_id` for a user from Servicenow.

To get the desired `sys_id` from Servicenow:
1. In the Application Navigator (left sidebar navigation menu) navigate to System Security > Users and Groups > Users
2. Use the search box to search for the user you want.
3. In the user listing click on a user.
4. In the user properties screen, click on the hamburger menu (three vertical bars) to the right of the left arrow in the upper left corner of the screen.
5. Click on 'Copy sys_id' to copy the `sys_id` directly to the clipboard, or click on Show XML to see the `sys_id` in a new window along with the rest of the user’s properties.

The `servicenow` report processor creates a ServiceNow incident based on a Puppet run report if at least one of the incident creation conditions are met.

By default the incident_creation_conditions are `['failures', 'corrective_changes']`. This means an incident will be created if there was a resource failure, a catalog compilation failure, or if a corrective change was made to a managed resource during a Puppet run. With these default parameters intentional changes such as adding a new resource to the catalog and Puppet bringing it into compliance for the first time, and pending changes, like running an agent in noop mode and the agent reporting changes would have occurred, will not result in an incident in Servicenow. If you would like to report on those types of changes `intentional_changes`, `pending_corrective_changes`, and `pending_intentional_changes` are also available as values for this parameter. As a shortcut you can also specify `always`, or to temporarily stop the module from creating any incidents at all you can set the `incident_creation_conditions` parameter to `never`.

The incident_mangement class also lets you specify additional (but optional) incident fields like the `category`, `subcategory` and `assigned_to` fields via the corresponding `category`, `subcategory`, and `assigned_to` parameters, respectively. See the `REFERENCE.md` for the full details.

Each incident will include the following information provided by Puppet:
* __Caller__: The user specified by the `sys_id` given to the `caller_id` parameter of the incident_management class. This can be any user in Servicenow and doesn’t need to be the same as the user that creates the incident via the username/password or oauth_token parameters.
* __Urgency__: Defaults to 3 - Low. Configurable
* __Impact__: Defaults to 3 - Low. Configurable
* __State__: Defaults to New. Configurable
* __Short Description__: Contains the following information
  * __Status__: changed, unchanged, failed, pending changes
  * __Node__: the fqdn of the node the agent ran on
  * __Report Time__: The timestamp of the report to help find the report in the console
* __Description__: See the description section from event management above. Incidents will get the same description

## Troubleshooting

To verify that everything worked, trigger a Puppet run on one of the nodes in the `PE Master` node group then log into the node. Once logged in, run `sudo tail -n 60 /var/log/puppetlabs/puppetserver/puppetserver.log | grep servicenow`. You should see some output. If not, then the class is probably not being classified properly. Either the class is not being assigned to the Puppet server nodes at all, or there may be catalog compilation errors on those nodes with the provided parameter values. Please use GitHub issues to file any bugs you find during your troubleshooting.

If you try to use the module and it doesn't work because of certificate validation errors, you can use the `skip_certificate_validation` parameter to disable certificate validation. The connection will still be SSL encrypted, but no certificate validation will be performed, which opens up a risk of 'Man in the middle' attacks. But, if you are using an internally hosted Servicenow instance that uses an SSL certificate that has not been imported for trust on the Puppet server machine, this may be necessary. Cloud hosted instances of Servicenow typically use a certificate signed by a well know public certificate authority, in which case, this parameter is not necessary.

If the module starts to throw errors indicating failures due to HTTP timeouts, you can try to use the `http_read_timeout` and `http_write_timeout` parameters to extend the amount of time available to complete a Servicenow API request.

If the module throws an error that contains the following text:

`failed to validate the PE console url 'https://<puppetserver URL>' via the '/auth/favicon.ico' endpoint: SSL_connect returned=1 errno=0 state=error: certificate verify failed`

This is an indication that the settings validation script cannot validate the availability and correct address of the pe console due to a certificate validation error. Please use the `$pe_console_cert_validation` parameter to change the method of certificate validation. By default the script assumes that the console is using the PE generate self signed cert (`selfsigned`). If you are using a certificate generated by an alternate means you will need to use either `truststore` if the ca cert that signed the console cert has been imported into the PE Servers OS trust store, or you can use `none` if you would like to skip certificate validate entirely. The connection will
still be SSL encrypted, but the certificate will not be validated.

## Development

### Unit tests
To run the unit tests:

```
bundle install
bundle exec rake spec_prep spec
```

### Acceptance tests
The acceptance tests use puppet-litmus in a multi-node fashion. The nodes consist of a 'master' node representing the PE master (and agent), and a 'ServiceNow' node representing the ServiceNow instance. All nodes are stored in a generated `inventory.yaml` file (relative to the project root) so that they can be used with Bolt.

To setup the test infrastructure, use `bundle exec rake acceptance:setup`. This will:

* **Provision the master VM**
* **Setup PE on the VM**
* **Setup the mock ServiceNow instance.** This is just a Docker container on the master VM that mimics the relevant ServiceNow endpoints. Its code is contained in `spec/support/acceptance/servicenow`.
* **Install the module on the master**

Each setup step is its own task; `acceptance:setup`'s implementation consists of calling these tasks. Also, all setup tasks are idempotent. That means its safe to run them (and hence `acceptance:setup`) multiple times.

**Note:** You can run the tests on a real ServiceNow instance. To do so, make sure that you've installed the event management plugin in your Servicenow instance, deleted the Servicenow portion of your inventory, and set the SN_INSTANCE, SN_USER, and SN_PASSWORD environment variables to appropriate values.

For example...
```
export SN_INSTANCE=dev84270.service-now.com
export SN_PASSWORD='d0hPFGhj5iNU!!!'
export SN_USER=admin
```

To run the tests after setup, you can do `bundle exec rspec spec/acceptance`. To teardown the infrastructure, do `bundle exec rake acceptance:tear_down`.

Below is an example acceptance test workflow:

```
bundle exec rake acceptance:setup
bundle exec rspec spec/acceptance
bundle exec rake acceptance:tear_down
```

**Note:** Remember to run `bundle exec rake acceptance:install_module` whenever you make updates to the module code. This ensures that the tests run against the latest version of the module.

#### Debugging the acceptance tests
Since the high-level setup is separate from the tests, you should be able to re-run a failed test multiple times via `bundle exec rspec spec/acceptance/path/to/test.rb`.

**Note:** Sometimes, the modules in `spec/fixtures/modules` could be out-of-sync. If you see a weird error related to one of those modules, try running `bundle exec rake spec_prep` to make sure they're updated.

## Known Issues

- PE Console URL in Event/Incident description may not work in PE versions older than 2019.8