# Skill: PowerShell Runspace Pool Provisioning

## When to use

When you need to run PowerShell scriptblocks in parallel without the module-autoload
races of `ForEach-Object -Parallel`. Use an `InitialSessionState`-provisioned runspace
pool with `PowerShell.BeginInvoke`/`EndInvoke`.

## Key APIs

```powershell
# Build session state
$iss = [System.Management.Automation.Runspaces.InitialSessionState]::CreateDefault()

# Add .ps1 script files as startup scripts (NOT ImportPSModule - that is for .psm1 only)
[void]$iss.StartupScripts.Add("C:\path\to\script.ps1")

# Inject preference variables
[void]$iss.Variables.Add(
    [System.Management.Automation.Runspaces.SessionStateVariableEntry]::new(
        'ErrorActionPreference', 'Stop', ''
    )
)

# Create pool - 4-arg overload: (minRunspaces, maxRunspaces, InitialSessionState, PSHost)
# There is NO 3-arg (int, int, InitialSessionState) overload.
$pool = [System.Management.Automation.Runspaces.RunspaceFactory]::CreateRunspacePool(
    1, $MaxParallel, $iss, $Host
)
$pool.Open()

# Launch async
$ps = [System.Management.Automation.PowerShell]::Create()
$ps.RunspacePool = $pool
$null = $ps.AddScript({ ... }).AddParameters(@{ Key = $Value })
$ar = $ps.BeginInvoke()

# Collect
$results = $ps.EndInvoke($ar)  # returns PSDataCollection<PSObject>
$ps.Dispose()

$pool.Close(); $pool.Dispose()
```

## Gotchas

1. `StartupScripts.Add()` and `Variables.Add()` return a bool in PowerShell - always use
   `[void]` to suppress it or the values leak out of the containing function.

2. `ImportPSModule` is for `.psm1` modules only. For `.ps1` files use `StartupScripts`.

3. `BeginInvoke`/`EndInvoke` preserves `PSCustomObject` property graphs intact in-process.
   `ForEach-Object -Parallel` can flatten nested PSCustomObject properties.

4. Scripts with `[Parameter(Mandatory)]` params in the top-level `param()` block will fail
   as startup scripts (no args passed). Verify all scripts have `param ()` not mandatory params.

5. Available overloads for `CreateRunspacePool`:
   - `()` - default
   - `(InitialSessionState)` - single runspace, custom ISS
   - `(int, int)` - pool with default ISS
   - `(int, int, PSHost)` - pool with default ISS + host
   - `(int, int, InitialSessionState, PSHost)` - pool with custom ISS + host  (USE THIS)
   No `(int, int, InitialSessionState)` 3-arg overload exists.

## Reference implementation

`modules/shared/WorkerPool.ps1` - `New-WorkerSessionState` + `Invoke-RunspacePoolTools`
