# simulator

A lightweight Go package for simulating large numbers of concurrent "players" that perform weighted, state-dependent actions over discrete turns.  
Useful for generating realistic event streams (e.g. for load testing, observability pipelines, or product analytics) without a real backend.

## Features

- Configurable number of players and initial state distributions
- Weighted random action selection + per-action conditions
- Mutable player state (`map[string]any`)
- Emits structured `Event`s (turn, action name, player ID, attributes)
- Supports single-turn, multi-turn, and continuous simulation
- Optional OTLP receiver endpoint for telemetry export

## Installation

```bash
go get github.com/jimtang2/simulator
```

(For local development with the companion `simulator-actions` package, use a `go.work` file or a `replace` directive.)

## Quick Start

```go
package main

import (
	"context"
	"fmt"
	"time"

	"github.com/jimtang2/simulator"
	// blank-import extra actions if desired
	// _ "github.com/jimtang2/simulator-actions/cex"
)

func main() {
	s, err := simulator.NewSimulator("config.yaml")
	if err != nil {
		panic(err)
	}

	out := make(chan simulator.Event, 1024)
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	go func() {
		for e := range out {
			fmt.Println(e)
		}
	}()

	// Run continuously until context is cancelled
	errCh := make(chan error, 1)
	go s.Continuous(ctx, out, errCh)

	<-ctx.Done()
	fmt.Println("simulation finished")
}
```

### Minimal config.yaml

```yaml
action_per_turn: 100
time_between_turn: 50          # ms

players:
  count: 1000
  init_states:
    - name: brand_new
      weight: 30
      state:
        onboarded: false
        logged_in: false
    - name: active
      weight: 70
      state:
        onboarded: true
        logged_in: true

actions:
  onboarding:
    weight: 10
  login:
    weight: 30
  logout:
    weight: 20
```

## Core Concepts

#### Class Diagram: Structure and Ownership

```mermaid
classDiagram
    class Simulator {
        -Config cfg
        -[]Player players
        -int64 turn
        +NewSimulator(configPath) Simulator
        +NextTurn(ctx, out) error
        +NextNTurn(n, out) error
        +Continuous(ctx, out, errCh)
        +Now() int64
    }

    class Config {
        +int ActionPerTurn
        +int TimeBetweenTurn
        +string OTLPReceiverEndpoint
        +map[string]ActionConfig Actions
        +PlayersConfig Players
    }

    class Player {
        -int id
        -map[string]any state
        +State(key) any
        +StateBool(key) bool
        +StateString(key) string
        +StateInt(key) int
        +SetState(key, value)
    }

    class Action {
        <<interface>>
        +Name() string
        +Condition(Player, Simulator) bool
        +Exec(Player, Simulator) error
        +Attributes(Player, Simulator) map[string]any
    }

    class Event {
        +int64 Turn
        +string ActionName
        +int PlayerID
        +map[string]any Attributes
    }

    class ActionConfig {
        +int Weight
    }

    class PlayersConfig {
        +int Count
        +[]StateConfig InitStates
    }

    Simulator *-- Config : owns
    Simulator *-- Player : manages many
    Config *-- ActionConfig : weights named actions
    Config *-- PlayersConfig : player population setup
    Simulator ..> Action : selects and executes
    Action ..> Player : reads and mutates state
    Action ..> Simulator : evaluates context
    Simulator ..> Event : emits after success
    Event ..> Action : records name
    Event ..> Player : records ID
```

#### Sequence Diagram: One Simulated Action

```mermaid
sequenceDiagram
    autonumber
    participant C as Config
    participant S as Simulator
    participant P as Player
    participant A as Action
    participant E as Event
    participant O as Output channel / OTLP pipeline

    Note over C,S: NewSimulator loads Config and creates Players
    C-->>S: ActionPerTurn, TimeBetweenTurn, action weights, initial states
    S->>P: Initialize players from configured states

    loop Each turn / action slot
        S->>S: Select random Player and Action
        S->>S: Apply configured Action weight
        S->>A: Condition(P, S)
        alt Condition is false
            A-->>S: Skip action
        else Condition is true
            S->>A: Exec(P, S)
            A->>P: Read/update mutable state
            P-->>A: Updated state
            A-->>S: Success
            S->>A: Attributes(P, S)
            A-->>S: Event attributes
            S->>E: NewEvent(P, A, S)
            E-->>S: Turn, ActionName, PlayerID, Attributes
            S->>O: Emit Event
        end
    end
```

Built-in actions: `onboarding`, `login`, `logout`.  
Additional actions can be registered with `simulator.AddAction(...)` or via blank imports from companion packages.

## API Summary

```go
s, err := simulator.NewSimulator(cfgPath)

s.NextTurn(ctx, out)          // one turn
s.NextNTurn(n, out)           // n turns (with sleep)
s.Continuous(ctx, out, errCh) // runs until ctx cancelled
```

## License

MIT