Learning the terminology
============

## What is an agent?

An agent is a simple software program - requiring no installation - which connects to CALDERA in order to get instructions. It then executes the instructions and sends the results back to the CALDERA server.

CALDERA includes the following agents:
* **Sandcat (54ndc47)**:  CALDERA's default agent, A GoLang agent that communicates through the HTTP contact.
* **Manx**: A reverse-shell agent that communicates via the TCP contact
* **Ragdoll**: A python agent that communicates via the HTML contact

Plugins are included for the sandcat and manx agents.

Agents can be deployed through the GUI by navigating to *Campaigns -> agents ->* and clicking 'Click here to deploy an agent'. Select the proper agent, operating system and options then copy and paste the command in a terminal or command prompt of your desired host to deploy the agent.

Individual agents can be configured by clicking on the button under the 'PID' column for the agent.

## What is a group?

A group is a collection of agents connected to CALDERA. A group allows you to start an operation against multiple computers at the same time instead of one at a time.

When an agent is started, it automatically joins a generic group, my_group, but you can add or remove custom groups once an agent is connected. 

During an operation, if an agent laterally moves to another computer, it will automatically become associated with the same group as the agent which started it.

## What is an ability?

An ability is a specific ATT&CK technique implementation (procedure). Abilities are stored in YML format and are loaded into CALDERA each time it starts. 

All abilities are stored inside the Stockpile plugin (`plugins/stockpile/data/abilities`), along with profiles which use them. 

Here is a sample ability:
```
- id: 9a30740d-3aa8-4c23-8efa-d51215e8a5b9
  name: Scan WIFI networks
  description: View all potential WIFI networks on host
  tactic: discovery
  technique:
    attack_id: T1016
    name: System Network Configuration Discovery
  platforms:
    darwin:
      sh:
        command: |
          ./wifi.sh scan
        payload: wifi.sh
    linux:
      sh:
        command: |
          ./wifi.sh scan
        payload: wifi.sh
    windows:
      psh,pwsh:
        command: |
          .\wifi.ps1 -Scan
        payload: wifi.ps1
```

Things to note:
* Each ability has a random UUID id
* Each ability requires a name, description, ATT&CK tactic and technique information
* Each ability requires a platforms list, which should contain at least 1 block for a supported operating system (platform). Currently, abilities can be created for darwin, linux or windows.
* Abilities can be added to an adversary through the GUI with the 'add ability' button

Bootstrap Abilities are abilities that run immediately after sending their first beacon in. A bootstrap ability can be added through the GUI by entering the ability id into the 'Bootstrap Abilities' field in the 'Agents' tab. Alternatively, you can edit the `conf/agents.yml` file and include the ability id in the bootstrap ability section of the file (ensure the server is turned off before editing any configuration files).

For each platform, there should be a list of executors. Currently Darwin and Linux platforms can use sh and Windows can use psh (PowerShell), cmd (command prompt) or pwsh (open-source PowerShell core).

Each platform block consists of a:
* command (required)
* payload (optional)
* cleanup (optional)
* parsers (optional)
* requirements (optional)

**Command**: A command can be 1-line or many and should contain the code you would like the ability to execute. The command can (optionally) contain variables, which are identified as #{variable}. In the example above, there is one variable used, #{files}. A variable means that you are letting CALDERA fill in the actual contents. CALDERA has a number of global variables:

* #{server} references the FQDN of the CALDERA server itself. Because every agent may know the location of CALDERA differently, using the #{server} variable allows you to let the system determine the correct location of the server.
* #{group} is the group a particular agent is a part of. This variable is mainly useful for lateral movement, where your command can start an agent within the context of the agent starting it. 
* #{paw} is the unique identifier - or paw print - of the agent.
* #{location} is the location of the agent on the client file system. 
* #{exe_name} is the executable name of the agent.

Global variables can be identified quickly because they will be single words.

You can use these global variables freely and they will be filled in before the ability is used. Alternatively, you can write in your own variables and supply CALDERA with facts to fill them in. 

**Payload**: A comma-separated list of files which the ability requires in order to run. In the windows executor above, the payload is wifi.ps1. This means, before the ability is used, the agent will download wifi.ps1 from CALDERA. If the file already exists, it will not download it. You can store any type of file in the payload directories of any plugin.

> Did you know that you can assign functions to execute on the server when specific payloads are requested for download? An example of this is the sandcat.go file. Check the plugins/sandcat/hook.py file to see how special payloads can be handled.

Payloads can be stored as regular files or you can xor (encode) them so the anti-virus on the server-side does not pick them up. To do this, run the app/utility/payload_encoder.py against the file to create an encoded version of it. Then store and reference the encoded payload instead of the original.

> The payload_encoder.py file has a docstring which explains how to use the utility.

**Cleanup**: An instruction that will reverse the result of the command. This is intended to put the computer back into the state it was before the ability was used. For example, if your command creates a file, you can use the cleanup to remove the file. Cleanup commands run after an operation, in the reverse order they were created. Cleaning up an operation is also optional, which means you can start an operation and instruct it to skip all cleanup instructions. 

Cleanup is not needed for abilities, like above, which download files through the payload block. Upon an operation completing, all payload files will be removed from the client (agent) computers.

**Parsers**: A list of parsing modules which can parse the output of the command into new facts. Interested in this topic? Check out [how CALDERA makes decisions](How-CALDERA-makes-decisions.md) which goes into detail about parsers. 

Abilities can also make use of two CALDERA REST API endpoints, file upload and download.

**Requirements**: Required relationships of facts that need to be established before this ability can be used.

## What is an adversary?

An adversary is a collection of abilities.

The abilities inside an adversary can optionally be grouped into phases, which allows a user to choose which order they are executed. During an operation, each phase of the adversary is run in order. If there are multiple abilities in the same phase, CALDERA will determine which order to run them, based on the information it has gathered thus far in the operation. This decision making process is known as the planner. The main reason to group abilities into phases is if an ability from a latter phase depends on the fact output from a previous phase.

An adversary can contain abilities which can be used on any platform (operating system). As an operation runs an adversary, CALDERA will match each ability to each agent and only send the matching ones to the agent.

CALDERA includes multiple pre-built adversaries. They are available through the GUI and YML profiles can be found in the `plugins/stockpile/data/adversaries` directory.

Adversaries can be built either through the GUI or by adding YML files into `plugins/stockpile/data/adversaries/` which is in the Stockpile plugin.

An adversary YML file can include a `phases` section that lists the IDs of the abilities to execute in each phase. Here is an example of such an adversary:
```
id: 5d3e170e-f1b8-49f9-9ee1-c51605552a08
name: Collection
description: A collection adversary pack
phases:
  1:
    - 1f7ff232-ebf8-42bf-a3c4-657855794cfe #find company emails
    - d69e8660-62c9-431e-87eb-8cf6bd4e35cf #find ip addresses
    - 90c2efaa-8205-480d-8bb6-61d90dbaf81b #find sensitive files
    - 6469befa-748a-4b9c-a96d-f191fde47d89 #create staging dir
```

An adversary can be included in another adversary as a pack of abilities. This can be used to organize ability phases into groups for reuse by multiple adversaries. To do so, put the ID of another adversary in a phase just like an ability. In this case, CALDERA will expand and complete all the phases of that adversary before moving to the next phase.

An adversary YML file can also contain a `packs` section that contains the IDs of other adversaries. The ability phases from these adversary packs will be merged together into any existing phases, whether from the `phases` section itself or from other adversaries in the `packs` section. Here is an example using packs without phases:
```
id: de07f52d-9928-4071-9142-cb1d3bd851e8
name: Hunter
description: Discover host details and steal sensitive files
packs:
  - 0f4c3c67-845e-49a0-927e-90ed33c044e0
  - 1a98b8e6-18ce-4617-8cc5-e65a1a9d490e
```

Adversary YML files must contain either `packs` or `phases`, or both.

## What is an operation?

An operation is started when you point an adversary at a group and have it run all capable abilities. 

An operation can be started with a number of optional configurations:

* **Group**: Which collection of agents would you like to run against
* **Adversary**: Which adversary profile would you like to run
* **Run immediately**: Run the operation immediately or start in a paused state
* **Planner**: You can select which logic library - or planner - you would like to use.
* **Fact source**: You can attach a source of facts to an operation. This means the operation will start with "pre-knowledge" of the facts, which it can use to fill in variables inside the abilities. 
* **Autonomous**: Run autonomously or manually. Manual mode will ask the operator to approve or discard each command.
* **Phases**: Run the adversary normally, abiding by phases, or smash all phases into a single one.
* **Auto-close**: Automatically close the operation when there is nothing left to do. Alternatively, keep the operation forever.
* **Obfuscators**: Select an obfuscator to encode each command with, before they are sent to the agents.
* **Jitter**: Agents normally check in with CALDERA every 60 seconds. Once they realize they are part of an active operation, agents will start checking in according to the jitter time, which is by default 2/8. This fraction tells the agents that they should pause between 2 and 8 seconds (picked at random each time an agent checks in) before using the next ability. 
* **Visibility**: How visible should the operation be to the defense. Defaults to 51 because each ability defaults to a visibility of 50. Abilities with a higher visibility than the operation visibility will be skipped.

## What is a fact?

A fact is an identifiable piece of information about a given computer. Facts are directly related to variables, which can be used inside abilities. 

Facts are composed of a:
* **trait**: a 3-part descriptor which identifies the type of fact. An example is host.user.name. A fact with this trait tells me that it is a user name. This format allows you to specify the major (host) minor (user) and specific (name) components of each fact.
* **value**: any arbitrary string. An appropriate value for a host.user.name may be "Administrator" or "John". 
* **score**: an integer which associates a relative importance for the fact. Every fact, by default, gets a score of 1. If a host.user.password fact is important or has a high chance of success if used, you may assign it a score of 5. When an ability uses a fact to fill in a variable, it will use those with the highest scores first. If a fact has a score of 0, it will be blacklisted - meaning it cannot be used in the operation.

> If a property has a major component = host (e.g., host.user.name) that fact will only be used by the host that collected it.

As hinted above, when CALDERA runs abilities, it scans the command and cleanup instructions for variables. When it finds one, it then looks at the facts it has and sees if it can replace the variables with matching facts (based on the property). It will then create new variants of each command/cleanup instruction for each possible combination of facts it has collected. Each variant will be scored based on the cumulative score of all facts inside the command. The highest scored variants will be executed first. 

Facts can be added or modified through the GUI by navigating to *Advanced -> Sources* and clicking on '+ add row'. 

## What is a source?

A source is a collection of facts that you have grouped together. A fact source can be applied to an operation when you start it, 
which gives the operation facts to fill in variables with. 

Sources can be added or modified through the GUI by navigating to *Advanced -> Sources*. 

## What is a rule?

A Rule is a way of restricting or placing boundaries on CALDERA. Rules are directly related to facts and should be included in a fact sheet.

Rules act similar to firewall rules and have three key components: fact, action, and match
1. **Fact** specifies the name of the fact that the rule will apply to.
2. **Action** (ALLOW,DENY) will allow or deny the fact from use if it matches the rule.
3. **Match** regex rule on a fact's value to determine if the rule applies.

During an operation, the planning service matches each link against the rule-set, discarding it if any of the fact assignments in the link match a rule specifying DENY and keeping it otherwise. In the case that multiple rules match the same fact assignment, the last one listed will be given priority.

**Example**
```
rules:
  - action: DENY
    fact: file.sensitive.extension
    match: .*
  - action: ALLOW
    fact: file.sensitive.extension
    match: txt
```
In this example only the txt file extension will be used. Note that the ALLOW action for txt supersedes the DENY for all, as the ALLOW rule is listed later in the policy. If the ALLOW rule was listed first, and the DENY rule second, then all values (including txt) for file.sensitive.extension would be discarded.

### Subnets
Rules can also match against subnets.

**Subnet Example**
```
  - action: DENY
    fact: my.host.ip
    match: .*
  - action: ALLOW
    fact: my.host.ip
    match: 10.245.112.0/24
```
In this example, the rules would permit CALDERA to only operate within the 10.245.112.1 to 10.245.112.254 range.

Rules can be added or modified through the GUI by navigating to *Advanced -> Sources* and clicking on '+ view rules'.

## What is a planner?

A planner is a module within CALDERA which contains logic for how a running operation should make decisions about which abilities to use and in what order.

Planners are single module Python files. Planners utilize the core system’s planning_svc.py, which has planning logic useful for various types of planners.

### The Batch planner

CALDERA ships with a default planner, _batch_. The _batch_ planner will retrieve all ability commands available and applicable for the operation and send them to the agents found in the operation's group. The _batch_ planner uses the planning service to retrieve ability commands based on the chosen advsersary and known agents in the operation. The abilities returned to the _batch_ planner are based on the agent matching the operating system (execution platform) of the ability and the ability command having no unsatisfied facts. The _batch_ planner will then send these ability commands to the agents and wait for them to be completed. After each batch of ability commands is completed, the _batch_ planner will again attempt to retrieve all ability commands available for the operation and attempt to repeat the cycle. This is required as once ability commands are executed, new additional ability commands may also become unlocked; e.g. required facts being present now, newly spawned agents etc... 

The _batch_ planner can be found in the ```mitre/stockpile``` github repository at ```app/batch.py```

### Custom Planners

For any other planner behavior and functionality, a custom planner is required. CALDERA has open sourced some custom planners, to include the _atomic_ and _buckets_ planners. From time to time, the CALDERA team will open source further planners as they become more widely used, publicly available etc...

The _atomic_ planner operates by atomically sending a single ability command to each agent in the operation's group at a time, progressing through abilities as they are enumerated in the underyling adversary profile. When a new agent is added to the operation, the _atomic_ planner will start with the first ability in the adversary profile.

The _buckets_ planner is example planner to demonstrate how to build a custom planner as well as the planning service utilities available to planners to aid in the formation decision logic.

The _batch_ and _buckets_ planners can be found in the ```mitre/stockpile``` github repository at ```app/batch.py``` and ```app/buckets.py```.

See [How to Build Planners](How-to-Build-Planners.md) for full walkthrough of how to build a custom planner and incorporate any custom decision logic that is desired.

## What is a plugin?

CALDERA is built using a plugin architecture on top of the core system. Plugins are separate git repositories that plug new features into the core system. Each plugin resides in the plugins directory and is loaded into CALDERA by adding it to the default.yml file.

Each plugin contains a single hook.py file in its root directory. This file should contain an initialize function, which gets called automatically for each loaded plugin when CALDERA boots. The initialize function contains the plugin logic that is getting "plugged into" the core system. This function takes a single parameter:

- **services**: a list of core services that live inside the core system. 

A plugin can add nearly any new functionality/features to CALDERA by using the two objects above. 

A list of plugins included with CALDERA can be found on the [Plugin library](Plugin-library.md) page.
