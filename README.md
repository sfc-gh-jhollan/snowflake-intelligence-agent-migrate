# Migrating From Table-based Agents to Agent Objects

Author/Owner: [Vishwal P K](mailto:vishwal.pk@snowflake.com) [Jeff Hollan](mailto:jeff.hollan@snowflake.com)  
Created: Jun 27, 2025 Last updated: Jul 8, 2025

# Introduction

Users who joined the Snowflake Intelligence preview prior to July 10 were asked to create a table to store agent configuration (`snowflake_intelligence.agents.config`). We are now transitioning to having agents be a first class object within Snowflake (`create agent`). As part of this, users who were active in the PrPr previously who wish to migrate agents from the “legacy” table to the new “native” object will need to take some steps to migrate.

Until July 21 we will support both “legacy” and “native” agents in the UI. But as of July 21, Snowflake Intelligence will start to only list agents in the new native format. Your old agents will still exist in the config table, and you can still migrate them after this date, but you will no longer see them listed in the agent dropdown after this date.

In order to facilitate a smooth migration from table-based agents to agent objects, we want to provide a sql command/script that automatically migrates table-based agents to agent objects. This doc shares the script and discusses relevant callouts/caveats/permissions.

# Before migrating

## Enable Agent Objects in Snowflake \- work with your SE/AE if questions on how to set account parameters

```sql
ALTER ACCOUNT <account_id>
SET FEATURE_CORTEX_AGENT_ENTITY = 'ENABLED'
  CORTEX_REST_DATA_AGENT_API_ENABLE = true
PARAMETER_COMMENT = 'Enable Agent Objects in Snowflake'
;
```

# Know before you run the migrations

## Required Privileges

Prior to running the script, please ensure the runner has the following privileges:

* Usage on the schema (`SNOWFLAKE_INTELLIGENCE.AGENTS`)  
* Create agent on the schema where you want the objects to be created.  
* Privileges to read all the rows in the table

## Creating With The Right Privileges

The user running the SQL command will be creating the agent objects with their active execution role and privileges, so ownership privileges will be automatically assigned to the role with which the script is run, so make sure to either select the right role or change grants after creating the objects using

```sql
// for granting privileges on the agentGRANT <PRIVILEGE> ON AGENT <AGENT_NAME> TO ROLE <ROLE_NAME>;
// for revoking privileges on the agentREVOKE <PRIVILEGE> ON AGENT <AGENT_NAME> FROM ROLE <ROLE_NAME>;
```

## Agent creation

### Location of SI agents

By default, agents are created in the `SNOWFLAKE_INTELLIGENCE.AGENTS` schema. Snowflake Intelligence currently only supports use of agent objects created in that specific schema.

* Note that we are working towards removing this schema requirement for Snowflake Intelligence, so this restriction will be removed in the future (\~Q4)

### Agent names

When creating objects, the script creates the objects with the same name specified in the `agent_name` column for table-based agents.

This means that if you have special characters in the name, the created object will follow the [Snowflake identifier rules](https://docs.snowflake.com/en/sql-reference/identifiers-syntax) and be enclosed in double quotes. For example, if the agent name in the table is `Demo Agent`, the path of the created object would be `snowflake_intelligence.agents.”Demo Agent”`.

# Migration

When ready to migrate, the following SQL command can be run to migrate table-based agents to agent objects.

```sql
WITH migrate_to_agent_objects AS PROCEDURE ()
    RETURNS ARRAY
    LANGUAGE JAVASCRIPT
    AS
    $$
        // configure database and schema
        var agentsDatabase = 'SNOWFLAKE_INTELLIGENCE';
        var agentsSchema = 'AGENTS';

        // retrieve table-based agents
        var select_table_agents_command = `
            SELECT
                agent_name,
                agent_description,
                grantee_roles,
                tools,
                tool_resources,
                response_instruction,
                sample_questions,
            FROM snowflake_intelligence.agents.config
        `;
        var table_agents_statement = snowflake.createStatement({ sqlText: select_table_agents_command });
        var table_agents_results = table_agents_statement.execute();
        var agents = [];
        while (table_agents_results.next())  {
            var instructions = (table_agents_results.getColumnValue(6) || '').split('[ORCHESTRATION_INSTRUCTION]');
            var response_instruction = instructions[0];
            var orchestration_instruction = instructions.length > 1 ? instructions[1] : '';
            agents.push({
                agent_name: table_agents_results.getColumnValue(1),
                agent_description: table_agents_results.getColumnValue(2) || '',
                grantee_roles: table_agents_results.getColumnValue(3) || [],
                tools: table_agents_results.getColumnValue(4) || [],
                tool_resources: table_agents_results.getColumnValue(5) || {},
                response_instruction,
                orchestration_instruction,
                sample_questions: table_agents_results.getColumnValue(7) || [],
            });
        }

        agents.forEach((agent) => {
            var models = {
                orchestration: 'auto',
            };
            var instructions = {
                ...(agent.response_instruction && { response: agent.response_instruction }),
                ...(agent.orchestration_instruction && { orchestration: agent.orchestration_instruction }),
                ...(agent.sample_questions.length && {
                    sample_questions: agent.sample_questions.map(question => ({ question: question.text }))
                }),
            };
            var spec = JSON.stringify({
              models,
              instructions,
              tools: agent.tools,
              tool_resources: agent.tool_resources,
            })

            // create agent object
            var create_object_command = `
                CREATE AGENT IDENTIFIER(:1)
                WITH PROFILE=:2
                    COMMENT=:3
                FROM SPECIFICATION $\$
                ${spec}
                $\$
            `;
            var agentPath = `${agentsDatabase}.${agentsSchema}."${agent.agent_name}"`;

            var binds = [
                agentPath,
                JSON.stringify({ display_name: agent.agent_name }),
                agent.agent_description,
            ];
            snowflake.createStatement({ sqlText: create_object_command, binds }).execute();

            // assign relevant grants
            agent.grantee_roles.map(grantee => {
                var grant_usage_command = `
                    GRANT USAGE
                    ON AGENT IDENTIFIER(:1)
                    TO ROLE IDENTIFIER(:2)
                `;
                snowflake.createStatement({ sqlText: grant_usage_command, binds: [agentPath, grantee] }).execute();
            })
        });
        
        return agents;
    $$
    CALL migrate_to_agent_objects();
```

After running the migrations, agent objects can be queried using to validate the execution.

```sql
// List all agentsSHOW AGENTS IN SCHEMA snowflake_intelligence.agents;
// Describe a specific agentDESCRIBE AGENT snowflake_intelligence.agents.<AGENT_NAME>;
```

# After Migrating

## Enable Agent Objects In Snowflake Intelligence

Once the agent objects are created, the corresponding customer/support engineers, can run the following param enablement command for the corresponding customer account:

```sql
ALTER ACCOUNT <account_id>
SET _DP_PEP_SI_ADMIN = true
  _DP_PEP_SI_ADMIN_CHAT = true
  _DP_PEP_SI_ENABLE_AGENT_OBJECTS = true
PARAMETER_COMMENT = 'Enable Agent Objects in Snowflake Intelligence'
;
```

## Cleanup

Once the customer has fully been moved to using agent objects (i.e., after script execution and enabling the parameter), table-based agents in `SNOWFLAKE_INTELLIGENCE.AGENTS.CONFIG` will no longer be listed in the Snowflake Intelligence UI.

Once the customer confirms successful migration, the config table can optionally be dropped. Please note because this is an irreversible action, we recommend only dropping the table after a complete move to using agent objects.

