# UI State and Components System

## Overview

The UI State and Components System allows you to create persistent, interactive UI components that display and manipulate structured data in the email agent. This system separates **state management** (data) from **rendering** (presentation), enabling actions and listeners to update state while components visualize it.

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Frontend (React)                   │
│  ┌──────────────┐  ┌──────────────┐                │
│  │  Component   │  │  Component   │                │
│  │  Renderer    │  │  Registry    │                │
│  └──────┬───────┘  └──────┬───────┘                │
│         │                  │                         │
│         └──────────────────┴───── WebSocket ────────┤
│                                                      │
└──────────────────────────────────────────────────────┘
                             │
┌────────────────────────────┼──────────────────────────┐
│                    Backend                           │
│  ┌─────────────────────────▼───────────────────┐    │
│  │        WebSocketHandler                     │    │
│  │  (Broadcasts state updates & components)    │    │
│  └─────────────────┬───────────────────────────┘    │
│                    │                                 │
│  ┌─────────────────▼──────┐  ┌──────────────────┐  │
│  │   UIStateManager       │  │ ComponentManager │  │
│  │  (Template discovery   │  │  (Instance mgmt) │  │
│  │   Hot reload, DB ops)  │  │                  │  │
│  └─────────────────┬──────┘  └──────────────────┘  │
│                    │                                 │
│  ┌─────────────────▼──────────────────────────┐    │
│  │          DatabaseManager                   │    │
│  │  (Persistent storage in SQLite)            │    │
│  └────────────────────────────────────────────┘    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Create a UI State Template

Define your data structure in `agent/custom_scripts/ui-states/`:

```typescript
// agent/custom_scripts/ui-states/my-state.ts
import type { UIStateTemplate } from '../types';

interface MyState {
  items: Array<{ id: string; name: string }>;
  count: number;
}

export const config: UIStateTemplate<MyState> = {
  id: 'my_state',
  name: 'My State',
  description: 'Description of what this state represents',
  initialState: {
    items: [],
    count: 0
  }
};
```

### 2. Create a React Component

Create a component in `client/components/custom/`:

```typescript
// client/components/custom/MyComponent.tsx
import React from 'react';
import type { ComponentProps } from './ComponentRegistry';

interface MyState {
  items: Array<{ id: string; name: string }>;
  count: number;
}

export const MyComponent: React.FC<ComponentProps<MyState>> = ({ state, onAction }) => {
  return (
    <div className="bg-white rounded-lg border p-4">
      <h2 className="text-lg font-bold mb-2">My Component</h2>
      <div>Count: {state.count}</div>
      <div>
        {state.items.map(item => (
          <div key={item.id}>{item.name}</div>
        ))}
      </div>
    </div>
  );
};
```

### 3. Register the Component

Add it to the `ComponentRegistry.ts`:

```typescript
// client/components/custom/ComponentRegistry.ts
import { MyComponent } from './MyComponent';

const componentRegistry = {
  'my_component': MyComponent,
  'task_board': TaskBoard,
  // ... other components
};
```

### 4. Create an Action to Update State

```typescript
// agent/custom_scripts/actions/update-my-state.ts
import type { ActionTemplate, ActionContext, ActionResult } from '../types';
import type { MyState } from '../ui-states/my-state';

export const config: ActionTemplate = {
  id: 'update_my_state',
  name: 'Update My State',
  description: 'Update the state and show component',
  icon: '📝',
  parameterSchema: {
    type: 'object',
    properties: {
      name: { type: 'string', description: 'Item name' }
    },
    required: ['name']
  }
};

export async function handler(
  params: { name: string },
  context: ActionContext ): Promise<ActionResult> {
  const stateId = 'my_state';

  // Get current state
  let state = await context.uiState.get<MyState>(stateId);

  if (!state) {
    state = { items: [], count: 0 };
  }

  // Update state
  state.items.push({
    id: `item_${Date.now()}`,
    name: params.name
  });
  state.count++;

  // Save state
  await context.uiState.set(stateId, state);

  // Return component to render
  return {
    success: true,
    message: `Added "${params.name}"`,
    components: [{
      instanceId: `comp_${Date.now()}`,
      componentId: 'my_component',
      stateId
    }]
  };
}
```

## Example: Task Board

A complete Task Board implementation is included:

- **State**: `agent/custom_scripts/ui-states/task-board.ts`
- **Component**: `client/components/custom/TaskBoard.tsx`
- **Actions**:
  - `create-task.ts` - Create new tasks
  - `update-task-status.ts` - Move tasks between columns

### Usage

1. **Create a task manually**:
   ```
   Ask the agent: "Create a task to review the quarterly report, high priority, due next Friday"
   ```

2. **Tasks appear automatically from emails**:
   - The system can extract action items from emails
   - Tasks link back to source emails
   - Organized in Kanban columns (To Do, In Progress, Done)

## API Reference

### UI State Operations (Backend)

#### In Actions/Listeners Context

```typescript
// Get UI state
const state = await context.uiState.get<MyState>(stateId);

// Set/update UI state
await context.uiState.set(stateId, newState);
```

### Component Props (Frontend)

```typescript
interface ComponentProps<T> {
  state: T;                    // Current UI state data
  onAction: (                  // Trigger action from component
    actionId: string,
    params: Record<string, any>
  ) => void;
}
```

### HTTP Endpoints

- `GET /api/ui-state/:stateId` - Fetch UI state
- `PUT /api/ui-state/:stateId` - Update UI state
- `DELETE /api/ui-state/:stateId` - Delete UI state
- `GET /api/ui-states` - List all UI states
- `GET /api/ui-state-templates` - List templates
- `GET /api/component-templates` - List component templates

### WebSocket Messages

**Server → Client:**
```typescript
// State updated
{
  type: 'ui_state_update',
  stateId: string,
  data: any
}

// Component instance to render
{
  type: 'component_instance',
  instance: ComponentInstance,
  sessionId: string
}

// Initial templates on connect
{
  type: 'ui_state_templates',
  templates: UIStateTemplate[]
}

{
  type: 'component_templates',
  templates: ComponentTemplate[]
}
```

**Client → Server:**
```typescript
// Trigger action from component
{
  type: 'component_action',
  instanceId: string,
  actionId: string,
  params: Record<string, any>
}
```

## File Structure

```
agent/custom_scripts/
├── ui-states/           # UI state templates
│   └── task-board.ts
├── components/          # Component metadata (not React code)
│   └── (future: component configs)
├── actions/             # Actions that update UI state
│   ├── create-task.ts
│   └── update-task-status.ts
├── listeners/           # Listeners that update UI state
│   └── task-extractor-listener.ts
└── .logs/
    └── ui-states/       # State update logs

client/components/custom/
├── ComponentRegistry.ts  # Component registration
├── TaskBoard.tsx         # React components
└── ...

ccsdk/
├── ui-state-manager.ts   # Backend state management
├── component-manager.ts  # Backend component management
└── ...
```

## Features

✅ **Persistent State** - Data survives across sessions
✅ **Real-time Updates** - Changes propagate via WebSocket
✅ **Type Safe** - Full TypeScript support with generics
✅ **Hot Reload** - Templates reload automatically on file changes
✅ **Audit Trail** - All state changes logged to JSONL
✅ **Interactive Components** - Trigger actions from UI
✅ **Separation of Concerns** - State independent of presentation

## Best Practices

1. **Keep state flat** - Avoid deeply nested structures
2. **Use immutable updates** - Always replace entire state objects
3. **Add TypeScript interfaces** - Define strong types for your state
4. **Component naming** - Use descriptive, unique component IDs
5. **Error handling** - Always wrap state operations in try-catch
6. **Logging** - Use `context.log()` for debugging
7. **Initial state** - Always provide sensible defaults

## Troubleshooting

### Component not rendering
- Check ComponentRegistry.ts - is your component registered?
- Check browser console for errors
- Verify component ID matches between action and registry

### State not persisting
- Check database has ui_states table (run migration if needed)
- Verify state ID is consistent across get/set calls
- Check .logs/ui-states/ for state update logs

### WebSocket not updating
- Ensure WebSocket connection is active
- Check browser console for WebSocket errors
- Verify UIStateManager is initialized in server.ts

## Next Steps

- Explore the Task Board example
- Create your own UI state and component
- Add listeners to automatically update state from emails
- Build interactive dashboards for your email workflows

For detailed implementation, see `UI_STATE_COMPONENTS_SPEC.md`.
