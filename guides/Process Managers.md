# Process managers

A process manager is responsible for coordinating one or more aggregates. It handles events and dispatches commands in response. You can think of a process manager as the opposite of an aggregate: aggregates handle commands and create events; process managers handle events and create commands. Process managers have state that can be used to track which aggregates are being orchestrated.

Use the `Commanded.ProcessManagers.ProcessManager` macro in your process manager module and implement the callback functions defined in the behaviour: `interested?/1`, `handle/2`, `apply/2`, and `error/4`.

## `interested?/1`

The `interested?/1` function is used to indicate which events the process manager handles. The response is used to route the event to an existing instance or start a new process instance:

- `{:start, process_uuid}` - create a new instance of the process manager.
- `{:continue, process_uuid}` - continue execution of an existing process manager.
- `{:stop, process_uuid}` - stop an existing process manager, shutdown its process, and delete its persisted state.
- `false` - ignore the event.

You can return a list of process identifiers when a single domain event must be handled by multiple process instances.

## `handle/2`

A `handle/2` function can be defined for each `:start` and `:continue` tagged event previously specified. It receives the process manager's state and the event to be handled. It must return the commands to be dispatched. This may be none, a single command, or many commands.

The `handle/2` function can be omitted if you do not need to dispatch a command and are only mutating the process manager's state.

## `apply/2`

The `apply/2` function is used to mutate the process manager's state. It receives the current state and the domain event, and must return the modified state.

This callback function is optional, the default behaviour is to retain the process manager's current state.

## `error/3`

You can define an `c:error/3` callback function to handle any errors or exceptions during event handling or returned by commands dispatched from your process manager. The function is passed the error (e.g. `{:error, :failure}`), the failed event or command, and a failure context. See `Commanded.ProcessManagers.FailureContext` for details.

Use pattern matching on the error and/or failed event/command to explicitly handle certain errors, events, or commands. You can choose to retry, skip, ignore, or stop the process manager after a command dispatch error.

The default behaviour, if you don't provide an `c:error/3` callback, is to stop the process manager using the exact error reason returned from the event handler function or command dispatch. You should supervise your process managers to ensure they are restarted on error.

The `error/3` callback function must return one of the following responses depending upon the severity of error and how you choose to handle it:

- `{:retry, context}` - retry the failed command, provide a context map containing any state passed to subsequent failures. This could be used to count the number of retries, failing after too many attempts.

- `{:retry, delay, context}` - retry the failed command, after sleeping for the requested delay (in milliseconds). Context is a map as described in `{:retry, context}` above.

- `{:skip, :discard_pending}` - discard the failed command and any pending commands.

- `{:skip, :continue_pending}` - skip the failed command, but continue dispatching any pending commands.

- `{:continue, commands, context}` - continue dispatching the given commands. This allows you to retry the failed command, modify it and retry, drop it, or drop all pending commands by passing an empty list `[]`.

- `{:stop, reason}` - stop the process manager with the given reason.

## Supervision

Supervise you process managers to ensure they are restarted on error.

```elixir
defmodule Bank.Payments.Supervisor do
  use Supervisor

  def start_link(_arg) do
    Supervisor.start_link(__MODULE__, [], name: __MODULE__)
  end

  def init(_arg) do
    Supervisor.init(
      [
        Bank.Payments.TransferMoneyProcessManager
      ],
      strategy: :one_for_one
    )
  end
end
```

### Error handling example

Define an `error/3` callback function to determine how to handle errors during event handling and command dispatch.

```elixir
defmodule ExampleProcessManager do
  use Commanded.ProcessManagers.ProcessManager,
    name: "ExampleProcessManager",
    router: ExampleRouter

  # Stop process manager after three failures
  def error({:error, _failure}, _failed_command, %{context: %{failures: failures}})
    when failures >= 2
  do
    {:stop, :too_many_failures}
  end

  # Retry command, record failure count in context map
  def error({:error, _failure}, _failed_command, %{context: context}) do
    context = Map.update(context, :failures, 1, fn failures -> failures + 1 end)
    {:retry, context}
  end
end
```

The default behaviour if you don't provide an `error/3` callback is to stop the process manager using the same error reason returned from the failed command dispatch.

## Example process manager

```elixir
defmodule TransferMoneyProcessManager do
  use Commanded.ProcessManagers.ProcessManager,
    name: "TransferMoneyProcessManager",
    router: BankRouter

  @derive Jason.Encoder
  defstruct [
    :transfer_uuid,
    :debit_account,
    :credit_account,
    :amount,
    :status
  ]

  def interested?(%MoneyTransferRequested{transfer_uuid: transfer_uuid}), do: {:start, transfer_uuid}
  def interested?(%MoneyWithdrawn{transfer_uuid: transfer_uuid}), do: {:continue, transfer_uuid}
  def interested?(%MoneyDeposited{transfer_uuid: transfer_uuid}), do: {:stop, transfer_uuid}
  def interested?(_event), do: false

  def handle(%TransferMoneyProcessManager{}, %MoneyTransferRequested{transfer_uuid: transfer_uuid, debit_account: debit_account, amount: amount}) do
    %WithdrawMoney{account_number: debit_account, transfer_uuid: transfer_uuid, amount: amount}
  end

  def handle(%TransferMoneyProcessManager{transfer_uuid: transfer_uuid, credit_account: credit_account, amount: amount}, %MoneyWithdrawn{}) do
    %DepositMoney{account_number: credit_account, transfer_uuid: transfer_uuid, amount: amount}
  end

  # State mutators

  def apply(%TransferMoneyProcessManager{} = transfer, %MoneyTransferRequested{transfer_uuid: transfer_uuid, debit_account: debit_account, credit_account: credit_account, amount: amount}) do
    %TransferMoneyProcessManager{transfer |
      transfer_uuid: transfer_uuid,
      debit_account: debit_account,
      credit_account: credit_account,
      amount: amount,
      status: :withdraw_money_from_debit_account
    }
  end

  def apply(%TransferMoneyProcessManager{} = transfer, %MoneyWithdrawn{}) do
    %TransferMoneyProcessManager{transfer |
      status: :deposit_money_in_credit_account
    }
  end
end
```

The name given to the process manager *must* be unique. This is used when subscribing to events from the event store to track the last seen event and ensure they are only received once.

```elixir
{:ok, _} = TransferMoneyProcessManager.start_link(start_from: :current)
```

You can choose to start the process router's event store subscription from the `:origin`, `:current` position or an exact event number using the `start_from` option. The default is to use the origin so it will receive all events. You typically use `:current` when adding a new process manager to an already deployed system containing historical events.

Process manager instance state is persisted to storage after each handled event. This allows the process manager to resume should the host process terminate.

## Event handling timeout

You can configure a timeout for event handling to ensure that events are processed in a timely manner without getting stuck.

An `event_timeout` option, defined in milliseconds, may be provided when using the `Commanded.ProcessManagers.ProcessManager` macro at compile time:

```elixir
defmodule TransferMoneyProcessManager do
  use Commanded.ProcessManagers.ProcessManager,
    name: "TransferMoneyProcessManager",
    router: BankRouter,
    event_timeout: :timer.minutes(10)

end
```

Or may be configured when starting a process manager:

```elixir
{:ok, _} = TransferMoneyProcessManager.start_link(event_timeout: :timer.hours(1))
```

After the timeout has elapsed, indicating the process manager has not processed an event within the configured period, the process manager is stopped. The process manager will be restarted if supervised and will retry the event, this should help resolve transient problems.
