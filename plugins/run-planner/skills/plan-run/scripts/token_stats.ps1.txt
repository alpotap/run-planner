param(
  # The project root. Default: two levels above this script (tools/run-planner/ -> project).
  [string]$Repo = ""
)

# run-planner: token statistics across every run, with a trend check. run_plan.ps1 calls it after
# its end-of-run summary; it can also be run on its own: .\tools\run-planner\token_stats.ps1
#
# Reads .runlogs/<run>/usage.csv (one row per model call, written by run_task.ps1) and the root
# .runlogs/usage.csv for runs that never had a folder. Then:
#   - appends one snapshot row per run to .runlogs/token_history.csv (never rewrites it);
#   - regenerates .runlogs/token_statistics.md from the rows on disk, with totals, models,
#     per-task figures and a trend section that flags abrupt step changes between runs.
# Nothing is committed; .runlogs/ is gitignored.

$ErrorActionPreference = "Stop"
if (-not $Repo) { $Repo = (Resolve-Path (Join-Path $PSScriptRoot "..\..")).Path }
$logs = Join-Path $Repo ".runlogs"
$history = Join-Path $logs "token_history.csv"
$report = Join-Path $logs "token_statistics.md"
$step = 0.5   # a change of 50% or more between consecutive runs is flagged
$inv = [Globalization.CultureInfo]::InvariantCulture
$utf8 = New-Object Text.UTF8Encoding $false
$checks = @(
  @{ key = "cost_per_task"; label = "cost per task" },
  @{ key = "cache_read_per_call"; label = "cache read per call" },
  @{ key = "turns_per_call"; label = "turns per call" }
)

function Get-Num($x) {
  $v = 0.0
  if ([double]::TryParse([string]$x, [Globalization.NumberStyles]::Float, $inv, [ref]$v)) { return $v }
  return 0.0
}
function Get-Total($rows, $col) {
  $t = 0.0
  foreach ($r in $rows) { $t += Get-Num $r.$col }
  return $t
}
function Get-RunNumber($name) {
  if ($name -match "^(\d+)") { return [int]$Matches[1] }
  return 0
}
function Format-Num($n, $decimals) { ([double]$n).ToString("N$decimals", $inv) }
function Format-Millions($n) { (([double]$n) / 1e6).ToString("N2", $inv) + "M" }
function Format-Change($c) { (([double]$c) * 100).ToString("+0;-0;0", $inv) + "%" }
function Get-Change($before, $after) {
  if ($before) { return ($after - $before) / $before }
  return 0.0
}
function Get-Row($cells) { "| " + ($cells -join " | ") + " |" }

# Every run that has a usage.csv with rows, in run order.
function Get-Runs {
  $found = @()
  $rootCsv = Join-Path $logs "usage.csv"
  if (Test-Path $rootCsv) { $found += [pscustomobject]@{ name = "(no run folder)"; path = $rootCsv } }
  foreach ($dir in Get-ChildItem $logs -Directory -ErrorAction SilentlyContinue) {
    $p = Join-Path $dir.FullName "usage.csv"
    if (Test-Path $p) { $found += [pscustomobject]@{ name = $dir.Name; path = $p } }
  }
  $runs = @()
  foreach ($f in ($found | Sort-Object @{ Expression = { Get-RunNumber $_.name } }, name)) {
    $rows = @(Import-Csv $f.path)
    if ($rows.Count) { $runs += [pscustomobject]@{ name = $f.name; rows = $rows } }
  }
  return $runs
}

function Get-Summary($rows) {
  $rows = @($rows)
  $calls = $rows.Count
  $tasks = @($rows | ForEach-Object { $_.task } | Sort-Object -Unique).Count
  $s = [ordered]@{
    tasks = $tasks
    calls = $calls
    errors = @($rows | Where-Object { $_.end -eq "error" }).Count
    turns = Get-Total $rows "turns"
    minutes = Get-Total $rows "minutes"
    input = Get-Total $rows "input"
    output = Get-Total $rows "output"
    cache_read = Get-Total $rows "cache_read"
    cache_write = Get-Total $rows "cache_write"
    cost = Get-Total $rows "cost_usd"
  }
  $s.cost_per_task = if ($tasks) { $s.cost / $tasks } else { 0.0 }
  $s.cache_read_per_call = if ($calls) { $s.cache_read / $calls } else { 0.0 }
  $s.output_per_call = if ($calls) { $s.output / $calls } else { 0.0 }
  $s.turns_per_call = if ($calls) { $s.turns / $calls } else { 0.0 }
  return $s
}

# Append one snapshot row per run. The file keeps every earlier row; the header is written once.
function Add-Snapshots($runs, $stamp) {
  $snap = foreach ($r in $runs) {
    $s = Get-Summary $r.rows
    [pscustomobject][ordered]@{
      snapshot = $stamp
      run = $r.name
      tasks = $s.tasks
      calls = $s.calls
      errors = $s.errors
      turns = [math]::Round($s.turns)
      minutes = [math]::Round($s.minutes, 1)
      input = [math]::Round($s.input)
      output = [math]::Round($s.output)
      cache_read = [math]::Round($s.cache_read)
      cache_write = [math]::Round($s.cache_write)
      cost_usd = [math]::Round($s.cost, 2)
      cost_per_task = [math]::Round($s.cost_per_task, 3)
      cache_read_per_call = [math]::Round($s.cache_read_per_call)
      output_per_call = [math]::Round($s.output_per_call)
      turns_per_call = [math]::Round($s.turns_per_call, 2)
    }
  }
  $lines = @($snap | ConvertTo-Csv -NoTypeInformation)
  if (Test-Path $history) { $lines = @($lines | Select-Object -Skip 1) }
  [IO.File]::AppendAllLines($history, [string[]]$lines, $utf8)
}

# One entry per pair of consecutive runs: what changed and which changes are abrupt.
function Get-Trend($runs) {
  $out = @()
  $prev = $null
  foreach ($r in $runs) {
    $s = Get-Summary $r.rows
    if ($prev) {
      $flags = @()
      foreach ($c in $checks) {
        $change = Get-Change $prev.s[$c.key] $s[$c.key]
        if ([math]::Abs($change) -ge $step) {
          $kind = if ($change -lt 0) { "ABRUPT DROP" } else { "ABRUPT RISE" }
          $flags += "${kind}: $($c.label) $(Format-Change $change)"
        }
      }
      $dcost = Get-Change $prev.s.cost_per_task $s.cost_per_task
      $dcache = Get-Change $prev.s.cache_read_per_call $s.cache_read_per_call
      $flagText = if ($flags) { $flags -join "; " } else { "-" }
      $out += [pscustomobject]@{
        from = $prev.name; to = $r.name
        before = $prev.s.cost_per_task; after = $s.cost_per_task
        dcost = $dcost; dcache = $dcache; flags = $flagText
      }
    }
    $prev = @{ name = $r.name; s = $s }
  }
  return $out
}

function Get-TotalCells($name, $s) {
  return @($name, $s.tasks, $s.calls, $s.errors,
    (Format-Num $s.turns 0), (Format-Num $s.minutes 1),
    (Format-Num $s.input 0), (Format-Num $s.output 0),
    (Format-Millions $s.cache_read), (Format-Millions $s.cache_write),
    ('$' + (Format-Num $s.cost 2)))
}

function Build-Report($runs, $stamp) {
  $every = @($runs | ForEach-Object { $_.rows })
  $all = Get-Summary $every
  $L = New-Object System.Collections.Generic.List[string]
  $L.Add("# Token statistics for all runs")
  $L.Add("")
  $L.Add("Generated $stamp by run_plan.ps1 (token_stats.ps1) from .runlogs/<run>/usage.csv. Each refresh also appends a snapshot to .runlogs/token_history.csv. Cost is the cost_usd the runner recorded. Calls that ended in error are counted because their tokens were spent.")
  $L.Add("")
  $L.Add("## Totals per run")
  $L.Add("")
  $L.Add("| Run | Tasks | Model calls | Calls ending in error | Turns | Minutes | Input tokens | Output tokens | Cache read | Cache write | Cost (USD) |")
  $L.Add("|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|")
  foreach ($r in $runs) { $L.Add((Get-Row (Get-TotalCells $r.name (Get-Summary $r.rows)))) }
  $L.Add((Get-Row (Get-TotalCells "**All runs**" $all)))

  $L.Add("")
  $L.Add("## Efficiency per run")
  $L.Add("")
  $L.Add("| Run | Cost per task | Cache read per call | Output per call | Turns per call | Cost per turn |")
  $L.Add("|---|---:|---:|---:|---:|---:|")
  foreach ($r in $runs) {
    $s = Get-Summary $r.rows
    $cpt = if ($s.turns) { $s.cost / $s.turns } else { 0.0 }
    $L.Add((Get-Row @($r.name, ('$' + (Format-Num $s.cost_per_task 2)), (Format-Millions $s.cache_read_per_call),
      ((Format-Num ($s.output_per_call / 1e3) 1) + "k"), (Format-Num $s.turns_per_call 0), ('$' + (Format-Num $cpt 3)))))
  }

  $L.Add("")
  $L.Add("## Trend between consecutive runs")
  $L.Add("")
  $L.Add("A change of $(Format-Num ($step * 100) 0)% or more in cost per task, cache read per call or turns per call is flagged.")
  $L.Add("")
  $L.Add("| From | To | Cost per task before | Cost per task after | Cost per task change | Cache read per call change | Flags |")
  $L.Add("|---|---|---:|---:|---:|---:|---|")
  foreach ($t in (Get-Trend $runs)) {
    $L.Add((Get-Row @($t.from, $t.to, ('$' + (Format-Num $t.before 2)), ('$' + (Format-Num $t.after 2)),
      (Format-Change $t.dcost), (Format-Change $t.dcache), $t.flags)))
  }

  $L.Add("")
  $L.Add("## Models")
  $L.Add("")
  $L.Add("Rows written before the runner recorded a model show as not recorded.")
  $L.Add("")
  $L.Add("| Model | Calls | Input | Output | Cache read | Cost (USD) |")
  $L.Add("|---|---:|---:|---:|---:|---:|")
  $byModel = @{}
  foreach ($r in $every) {
    $key = if ($r.model) { $r.model } else { "not recorded" }
    if (-not $byModel.ContainsKey($key)) { $byModel[$key] = @() }
    $byModel[$key] += $r
  }
  foreach ($kv in ($byModel.GetEnumerator() | Sort-Object { -(Get-Total $_.Value "cost_usd") })) {
    $s = Get-Summary $kv.Value
    $L.Add((Get-Row @($kv.Key, $s.calls, (Format-Num $s.input 0), (Format-Num $s.output 0),
      (Format-Millions $s.cache_read), ('$' + (Format-Num $s.cost 2)))))
  }

  $L.Add("")
  $L.Add("## Per task")
  $L.Add("")
  $L.Add("| Run | Task | Calls | Turns | Minutes | Output tokens | Cache read | Cost (USD) | Ended in error |")
  $L.Add("|---|---|---:|---:|---:|---:|---:|---:|---:|")
  foreach ($r in $runs) {
    foreach ($g in ($r.rows | Group-Object task)) {
      $s = Get-Summary $g.Group
      $L.Add((Get-Row @($r.name, $g.Name, $s.calls, (Format-Num $s.turns 0), (Format-Num $s.minutes 1),
        (Format-Num $s.output 0), (Format-Millions $s.cache_read), ('$' + (Format-Num $s.cost 2)),
        $(if ($s.errors) { "yes" } else { "no" }))))
    }
  }
  return (($L -join "`n") + "`n")
}

$runs = @(Get-Runs)
if ($runs.Count -eq 0) {
  Write-Host "Token statistics: no usage.csv rows under .runlogs yet - nothing to count." -ForegroundColor DarkGray
  return
}
$stamp = Get-Date -Format "yyyy-MM-ddTHH:mm:ss"
Add-Snapshots $runs $stamp
[IO.File]::WriteAllText($report, (Build-Report $runs $stamp), $utf8)

$flagged = @(Get-Trend $runs | Where-Object { $_.flags -ne "-" })
Write-Host "Token statistics: $($runs.Count) run(s). Report: $report" -ForegroundColor Cyan
Write-Host "History appended: $history ($($runs.Count) rows, snapshot $stamp)" -ForegroundColor Cyan
if ($flagged.Count -eq 0) { Write-Host "No abrupt changes between consecutive runs." -ForegroundColor Green }
foreach ($t in $flagged) { Write-Host "  $($t.from) -> $($t.to): $($t.flags)" -ForegroundColor Yellow }
