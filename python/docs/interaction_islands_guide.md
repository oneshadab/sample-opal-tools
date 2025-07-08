# Creating Interaction Islands with opal_tools_sdk

This guide explains how to create interactive UI components (called "islands") using the `opal_tools_sdk`. Interaction islands allow users to interact with your tools through a visual interface rather than just programmatic calls.

## Prerequisites

Before using this guide, ensure you have:
- Python 3.8+ installed
- The `opal_tools_sdk` package installed
- Basic understanding of async/await in Python
- Familiarity with Pydantic for data validation

## Table of Contents

1. [Overview](#overview)
2. [Basic Structure](#basic-structure)
3. [Parameter Models](#parameter-models)
4. [Tool Decorator](#tool-decorator)
5. [Function Signature](#function-signature)
6. [Execution Modes](#execution-modes)
7. [Creating Islands](#creating-islands)
8. [Field Configuration](#field-configuration)
9. [Action Configuration](#action-configuration)
10. [Multiple Islands](#multiple-islands)
11. [Complete Examples](#complete-examples)
12. [Best Practices](#best-practices)

## Overview

Interaction islands provide a visual interface for tools that need user interaction. They're particularly useful when:
- Users need to review data before taking action
- Multiple related operations need to be performed
- Complex forms need to be filled out
- Users need to make decisions based on presented information

## Basic Structure

Every interaction island tool follows this basic pattern:

```python
from opal_tools_sdk.decorators import tool
from pydantic import BaseModel, Field

# These classes are part of the opal_tools_sdk framework
# AuthData: Contains authentication information
# Environment: Contains execution environment details
# IslandConfig: Configuration for UI islands
# IslandResponse: Response wrapper for islands
from opal_tools_sdk.types import AuthData, Environment, IslandConfig, IslandResponse

# 1. Define parameter model
class MyToolParams(BaseModel):
    field1: str = Field(description="Description of field1")
    field2: Optional[str] = Field(description="Description of field2", default=None)

# 2. Create the tool function
@tool(
    "my_tool_name",
    "Description of what this tool does",
    auth_requirements=[
        {"provider": "OptiID", "scope_bundle": "default", "required": True},
    ],
)
async def my_tool(
    params: MyToolParams,
    auth_data: AuthData,
    environment: Environment,
):
        # 3. Handle headless mode
    if environment["execution_mode"] == "headless":
        # In headless mode, perform the operation directly without UI
        return await perform_headless_operation(params, auth_data)

    # 4. Create interactive islands for UI mode
    actions = [
        IslandConfig.Action(
            name="action_name",
            label="Action Label",
            type="button",
            operation="create",  # or "update"
            endpoint="/interactions/endpoint_name",
        )
    ]

    island = IslandConfig(
        fields=[
            IslandConfig.Field(
                name="field_name",
                label="Field Label",
                type="string",
                value=params.field1,
            ),
            # ... more fields
        ],
        actions=actions,
    )

    return IslandResponse.create([island])
```

## Parameter Models

Define your input parameters using Pydantic models. These models define the structure and validation rules for your tool's input parameters:

```python
from typing import Optional
from pydantic import BaseModel, Field

class CreateCampaignParams(BaseModel):
    campaign_title: str = Field(description="The title of the campaign")
    campaign_brief: str = Field(description="The brief of the campaign")
    parent_campaign_id: Optional[str] = Field(
        description="The ID of the parent campaign",
        default=None
    )
```

**Field Types:**
- `str`: Text input
- `Optional[str]`: Optional text input (can be None)
- `list[str]`: List of strings
- `int`: Integer values
- `bool`: Boolean values
- `float`: Decimal numbers

## Tool Decorator

The `@tool` decorator configures your function as a tool and registers it with the opal_tools_sdk framework:

```python
@tool(
    "create_campaign",                    # Tool name (used internally)
    "Create campaign suggestions for CMP", # Human-readable description
    auth_requirements=[
        {"provider": "OptiID", "scope_bundle": "default", "required": True},
    ],
)
```

**Auth Requirements:**
- `provider`: Authentication provider (e.g., "OptiID", "OAuth2", "APIKey")
- `scope_bundle`: Required permissions or scopes
- `required`: Whether authentication is mandatory (True/False)

## Function Signature

Your tool function must have this exact signature to work with the opal_tools_sdk framework:

```python
async def my_tool(
    params: MyToolParams,      # Your parameter model (defined above)
    auth_data: AuthData,       # Authentication data provided by the framework
    environment: Environment,   # Execution environment details
):
```

## Execution Modes

Tools support two execution modes:

### Headless Mode
When `environment["execution_mode"] == "headless"`, the tool runs programmatically without UI. This is useful for automated workflows or when the tool is called from another system:

```python
if environment["execution_mode"] == "headless":
    # Perform the operation directly without creating UI components
    return await perform_operation(params, auth_data)
```

### Interactive Mode
When not in headless mode, create islands for user interaction. This is the main use case for interaction islands - providing a visual interface for users:

```python
# Create islands for interactive mode
actions = [...]
island = IslandConfig(...)
return IslandResponse.create([island])
```

## Creating Islands

Islands are UI components that users interact with. Each island consists of:

1. **Fields**: Data to display or collect from users
2. **Actions**: Buttons that trigger operations when clicked

```python
island = IslandConfig(
    fields=[
        # Your fields here
    ],
    actions=[
        # Your actions here
    ],
)

return IslandResponse.create([island])
```

## Field Configuration

Fields define the data displayed in the island:

```python
IslandConfig.Field(
    name="field_name",           # Internal identifier
    label="Field Label",         # User-visible label
    type="string",              # Field type: "string", "boolean", "json"
    value="field_value",        # Default/current value
    hidden=False,               # Whether to hide from user
    options=[],                 # For dropdown fields
)
```

**Field Types:**
- `"string"`: Text input/display (most common)
- `"boolean"`: True/false values (checkboxes)
- `"json"`: JSON data (for complex structured data)

**Common Patterns:**

**Display Field (shown to user):**
```python
IslandConfig.Field(
    name="key",
    label="Title",
    type="string",
    value=params.campaign_title,
)
```

**Hidden Field (for internal data, not shown to user):**
```python
IslandConfig.Field(
    name="campaign_id",
    label="Campaign ID",
    type="string",
    value=params.campaign_id,
    hidden=True,  # Hidden from user
)
```

**Conditional Fields (only shown when needed):**
```python
hidden_fields: list[IslandConfig.Field] = []
if params.parent_campaign_id:
    hidden_fields.append(
        IslandConfig.Field(
            name="parent_campaign_id",
            label="Parent Campaign ID",
            type="string",
            value=params.parent_campaign_id,
            hidden=True,
        )
    )

# Use in island
island = IslandConfig(
    fields=[
        *hidden_fields,  # Spread conditional fields
        # ... other fields
    ],
    # ...
)
```

## Action Configuration

Actions define what users can do with the island. Each action creates a button that users can click:

```python
IslandConfig.Action(
    name="action_name",         # Internal identifier
    label="Action Label",       # Button text (what user sees)
    type="button",             # Action type (usually "button")
    operation="create",         # Operation: "create", "update"
    endpoint="/interactions/endpoint_name",  # Backend endpoint to call
)
```

**Common Action Types:**

**Create Operation (for new items):**
```python
IslandConfig.Action(
    name="create_object",
    label="Create Campaign",
    type="button",
    operation="create",
    endpoint="/interactions/create_campaign",
)
```

**Update Operation (for existing items):**
```python
IslandConfig.Action(
    name="update_object",
    label="Update Campaign Brief",
    type="button",
    operation="update",
    endpoint="/interactions/update_campaign_brief",
)
```

## Multiple Islands

You can create multiple islands for complex workflows. This is useful when you need to show multiple related items or operations:

```python
islands = []

for title in params.titles:
    actions = [
        IslandConfig.Action(
            name="create_objects",
            label="Create Tasks",
            type="button",
            operation="create",
            endpoint="/interactions/create_task_for_campaign",
        )
    ]

    fields = [
        IslandConfig.Field(
            name="key",
            label="Task Title",
            type="string",
            value=title,
        ),
        IslandConfig.Field(
            name="campaign_id",
            label="Campaign ID",
            type="string",
            value=params.campaign_id,
            hidden=True,
        ),
        IslandConfig.Field(
            name="title",
            label="Task Title",
            type="string",
            value=title,
        ),
    ]

    island = IslandConfig(
        fields=fields,
        actions=actions,
    )
    islands.append(island)

return IslandResponse.create(islands)
```

## Complete Examples

### Example 1: Simple Create Operation

This example shows how to create a simple island for creating a new campaign:

```python
@tool(
    "create_campaign",
    "Create campaign suggestions for CMP",
    auth_requirements=[
        {"provider": "OptiID", "scope_bundle": "default", "required": True},
    ],
)
async def create_campaign(
    params: CreateCampaignParams,
    auth_data: AuthData,
    environment: Environment,
):
    # Handle headless mode
    if environment["execution_mode"] == "headless":
        # In headless mode, perform the operation directly
        return await perform_campaign_creation(params, auth_data)

    # Create interactive island
    hidden_fields: list[IslandConfig.Field] = []
    if params.parent_campaign_id:
        hidden_fields.append(
            IslandConfig.Field(
                name="parent_campaign_id",
                label="Parent Campaign ID",
                type="string",
                value=params.parent_campaign_id,
                hidden=True,
            )
        )

    actions = [
        IslandConfig.Action(
            name="create_object",
            label="Create Campaign",
            type="button",
            operation="create",
            endpoint="/interactions/create_campaign",
        )
    ]

    island = IslandConfig(
        fields=[
            *hidden_fields,
            IslandConfig.Field(
                name="key",
                label="Title",
                type="string",
                value=params.campaign_title,
            ),
            IslandConfig.Field(
                name="title",
                label="Title",
                type="string",
                value=params.campaign_title,
            ),
            IslandConfig.Field(
                name="brief",
                label="Brief",
                type="string",
                value=params.campaign_brief,
                hidden=True,
            ),
        ],
        actions=actions,
    )

    return IslandResponse.create([island])
```

### Example 2: Update Operation

This example shows how to create an island for updating existing data:

```python
@tool(
    "update_campaign_brief",
    "Update campaign brief for CMP",
    auth_requirements=[
        {"provider": "OptiID", "scope_bundle": "default", "required": True},
    ],
)
async def update_campaign_brief(
    params: UpdateCampaignBriefParams,
    auth_data: AuthData,
    environment: Environment,
):
    # Handle headless mode
    if environment["execution_mode"] == "headless":
        # In headless mode, perform the update directly
        return await perform_campaign_update(params, auth_data)

    # Create interactive island
    actions = [
        IslandConfig.Action(
            name="update_object",
            label="Update Campaign Brief",
            type="button",
            operation="update",
            endpoint="/interactions/update_campaign_brief",
        )
    ]

    island = IslandConfig(
        fields=[
            IslandConfig.Field(
                name="key",
                label="Campaign Brief",
                type="string",
                value=params.update_summary,
            ),
            IslandConfig.Field(
                name="campaign_id",
                label="Campaign ID",
                type="string",
                value=params.campaign_id,
                hidden=True,
            ),
            IslandConfig.Field(
                name="brief",
                label="Brief Content",
                type="string",
                value=params.brief_content,
            ),
            IslandConfig.Field(
                name="update_summary",
                label="Update Summary",
                type="string",
                value=params.update_summary,
                hidden=True,
            ),
        ],
        actions=actions,
    )

    return IslandResponse.create([island])
```

### Example 3: Complex Form with Multiple Fields

This example shows how to create an island with multiple fields for complex data entry:

```python
@tool(
    "create_article_in_task",
    "Create an article within a task in CMP",
    auth_requirements=[
        {"provider": "OptiID", "scope_bundle": "default", "required": True},
    ],
)
async def create_article_in_task(
    params: CreateArticleInTaskParams,
    auth_data: AuthData,
    environment: Environment,
):
    # Handle headless mode
    if environment["execution_mode"] == "headless":
        # In headless mode, perform the article creation directly
        return await perform_article_creation(params, auth_data)

    # Create interactive island
    actions = [
        IslandConfig.Action(
            name="create_object",
            label="Create Article",
            type="button",
            operation="create",
            endpoint="/interactions/create_task_article",
        )
    ]

    island = IslandConfig(
        fields=[
            IslandConfig.Field(
                name="key",
                label="Article Title",
                type="string",
                value=params.title,
            ),
            IslandConfig.Field(
                name="task_id",
                label="Task ID",
                type="string",
                value=params.task_id,
                hidden=True,
            ),
            IslandConfig.Field(
                name="title",
                label="Article Title",
                type="string",
                value=params.title,
            ),
            IslandConfig.Field(
                name="body",
                label="Article Body",
                type="string",
                value=params.body,
            ),
            IslandConfig.Field(
                name="meta_title",
                label="Meta Title",
                type="string",
                value=params.meta_title,
            ),
            IslandConfig.Field(
                name="meta_description",
                label="Meta Description",
                type="string",
                value=params.meta_description,
            ),
        ],
        actions=actions,
    )
    return IslandResponse.create([island])
```

## Best Practices

### 1. Always Handle Both Execution Modes
```python
# Always check execution mode first
if environment["execution_mode"] == "headless":
    # Handle programmatic execution
    return await perform_operation(params, auth_data)

# Handle interactive mode
# Create islands...
```

### 2. Use Descriptive Field Names and Labels
```python
# Good - clear and descriptive
IslandConfig.Field(name="campaign_title", label="Campaign Title", ...)

# Avoid - unclear abbreviations
IslandConfig.Field(name="ct", label="CT", ...)
```

### 3. Hide Internal Data from Users
```python
# Hide IDs and internal fields from users
IslandConfig.Field(
    name="campaign_id",
    label="Campaign ID",
    type="string",
    value=params.campaign_id,
    hidden=True,  # Users don't need to see this
)
```

### 4. Use Consistent Action Names
```python
# Create operations
name="create_object" or name="create_objects"

# Update operations
name="update_object" or name="update_objects"
```

### 5. Provide Clear Action Labels
```python
# Good - descriptive and specific
label="Create Campaign"
label="Update Campaign Brief"

# Avoid - generic and unclear
label="Submit"
label="OK"
```

### 6. Handle Optional Parameters
```python
hidden_fields: list[IslandConfig.Field] = []
if params.optional_field:
    hidden_fields.append(
        IslandConfig.Field(
            name="optional_field",
            label="Optional Field",
            type="string",
            value=params.optional_field,
            hidden=True,
        )
    )
```

### 7. Use Meaningful Endpoints
```python
# Good - descriptive and specific
endpoint="/interactions/create_campaign"
endpoint="/interactions/update_campaign_brief"

# Avoid - generic and unclear
endpoint="/api/v1/action"
endpoint="/process"
```

## Common Patterns

### Pattern 1: Create with Preview
Show users what will be created before they confirm:

```python
island = IslandConfig(
    fields=[
        IslandConfig.Field(
            name="key",
            label="Preview",
            type="string",
            value=f"Will create: {params.title}",
        ),
        # Hidden fields for the actual operation
        IslandConfig.Field(
            name="title",
            label="Title",
            type="string",
            value=params.title,
            hidden=True,
        ),
    ],
    actions=[
        IslandConfig.Action(
            name="create_object",
            label="Confirm Create",
            type="button",
            operation="create",
            endpoint="/interactions/create_item",
        )
    ],
)
```

### Pattern 2: Update with Current Values
Show current values and allow updates:

```python
island = IslandConfig(
    fields=[
        IslandConfig.Field(
            name="key",
            label="Current Value",
            type="string",
            value=f"Current: {current_value}",
        ),
        IslandConfig.Field(
            name="new_value",
            label="New Value",
            type="string",
            value=params.new_value,
        ),
    ],
    actions=[
        IslandConfig.Action(
            name="update_object",
            label="Update",
            type="button",
            operation="update",
            endpoint="/interactions/update_item",
        )
    ],
)
```

### Pattern 3: Multiple Related Operations
Create multiple islands for related operations:

```python
islands = []
for item in items:
    island = IslandConfig(
        fields=[
            IslandConfig.Field(
                name="key",
                label="Item",
                type="string",
                value=item.name,
            ),
            IslandConfig.Field(
                name="item_id",
                label="Item ID",
                type="string",
                value=item.id,
                hidden=True,
            ),
        ],
        actions=[
            IslandConfig.Action(
                name="process_item",
                label="Process",
                type="button",
                operation="create",
                endpoint="/interactions/process_item",
            )
        ],
    )
    islands.append(island)

return IslandResponse.create(islands)
```

This guide covers the essential patterns for creating interaction islands with the `opal_tools_sdk`. Use these patterns to build intuitive, user-friendly interfaces for your tools.

## Summary

Interaction islands provide a powerful way to create user-friendly interfaces for your tools. Key takeaways:

1. **Always handle both execution modes** - headless for automation, interactive for user interfaces
2. **Use descriptive names and labels** - make your islands intuitive for users
3. **Hide internal data** - only show users what they need to see
4. **Provide clear actions** - make it obvious what users can do
5. **Follow consistent patterns** - use the same structure across your tools

## Next Steps

To get started with interaction islands:

1. Install the `opal_tools_sdk` package
2. Set up your authentication requirements
3. Create your parameter models using Pydantic
4. Implement your tool function with both execution modes
5. Test your islands in both headless and interactive modes

For more advanced features, refer to the `opal_tools_sdk` documentation and explore the framework's additional capabilities.