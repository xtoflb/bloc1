# Test de charge du service Apache2
## 1. Autorisation de lancement d'un script PowerShell
Ouvrez PowerShell ISE sur votre poste et executez dans la console la commande :
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
## 2. Script de test de charge
Copiez - collez ce script dans la fenêtre d'édition
```powershell
# TestDeCharge.ps1
# Objectif : test de charge HTTP simultané par un groupe d'étudiants
#            1) vers un serveur web unique (Apache2)
#            2) vers un load balancer HAProxy avec plusieurs backends
# Usage pédagogique uniquement, dans un labo isolé.
#
# si besoin pour autoriser les scripts, entrez la commande ci-dessous en console PowerShell
# Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

param(
    [string]$TargetUrl = "http://172.17.2.230",     # ou http://172.17.2.235 pour le test HAProxy

    [ValidateSet("FixedCount","Duration")]
    [string]$Mode = "Duration",                     # "Duration" recommandé pour une charge soutenue

    [int]$ConcurrentSessions = 50,                   # nb de workers en parallèle (vrais threads, pas des process)
    [int]$TotalRequests = 3000,                      # utilisé seulement si Mode = FixedCount
    [int]$DurationSeconds = 60,                       # utilisé seulement si Mode = Duration
    [int]$TimeoutSec = 5,

    [string]$Label = "Test",                         # ex: "Apache-Direct" ou "HAProxy-4backends"
    [string]$ResultFile = "",

    [switch]$NoCache                                  # ajoute un paramètre aléatoire pour éviter tout cache
)

$ErrorActionPreference = "Stop"

# --- Réglages réseau .NET : lever la limite de connexions simultanées par hôte ---
[System.Net.ServicePointManager]::DefaultConnectionLimit = [Math]::Max(1000, $ConcurrentSessions * 4)
[System.Net.ServicePointManager]::Expect100Continue = $false
[System.Net.ServicePointManager]::UseNagleAlgorithm = $false

if (-not $ResultFile) {
    $ts = Get-Date -Format "yyyyMMdd_HHmmss"
    $ResultFile = "resultats_${Label}_$ts.csv"
}

Write-Host "=== Test de charge : $Label ===" -ForegroundColor Cyan
Write-Host "Cible            : $TargetUrl"
Write-Host "Mode             : $Mode"
Write-Host "Sessions //      : $ConcurrentSessions"
if ($Mode -eq "FixedCount") { Write-Host "Requêtes totales : $TotalRequests" }
else { Write-Host "Durée            : $DurationSeconds s" }
Write-Host ""

# --- Collection thread-safe pour les résultats ---
$results = [System.Collections.Concurrent.ConcurrentBag[psobject]]::new()

# --- Pool de runspaces : léger, tourne dans le process courant ---
$sessionState = [System.Management.Automation.Runspaces.InitialSessionState]::CreateDefault()
$pool = [runspacefactory]::CreateRunspacePool(1, $ConcurrentSessions, $sessionState, $Host)
$pool.Open()

$scriptBlock = {
    param($url, $timeoutSec, $noCache, $results, $mode, $totalRequests, $endTime)

    while ($true) {
        if ($mode -eq "FixedCount") {
            if ($results.Count -ge $totalRequests) { break }
        } else {
            if ((Get-Date) -ge $endTime) { break }
        }

        $reqUrl = $url
        if ($noCache) {
            $sep = if ($url -match '\?') { '&' } else { '?' }
            $reqUrl = "$url$sep`_=$([guid]::NewGuid().ToString('N'))"
        }

        $sw = [System.Diagnostics.Stopwatch]::StartNew()
        try {
            $resp = Invoke-WebRequest -Uri $reqUrl -Method Get -UseBasicParsing -TimeoutSec $timeoutSec -Headers @{ "Cache-Control" = "no-cache" }
            $sw.Stop()
            $server = $null
            try { $server = $resp.Headers["X-Served-By"] } catch {}
            $results.Add([pscustomobject]@{
                Timestamp  = (Get-Date).ToString("HH:mm:ss.fff")
                StatusCode = [int]$resp.StatusCode
                DurationMs = [math]::Round($sw.Elapsed.TotalMilliseconds, 2)
                Success    = $true
                ServedBy   = $server
                ErrorMsg   = $null
            })
        }
        catch {
            $sw.Stop()
            $statusCode = $null
            if ($_.Exception.Response) {
                try { $statusCode = [int]$_.Exception.Response.StatusCode } catch {}
            }
            $results.Add([pscustomobject]@{
                Timestamp  = (Get-Date).ToString("HH:mm:ss.fff")
                StatusCode = $statusCode
                DurationMs = [math]::Round($sw.Elapsed.TotalMilliseconds, 2)
                Success    = $false
                ServedBy   = $null
                ErrorMsg   = $_.Exception.Message
            })
        }
    }
}

$endTime = (Get-Date).AddSeconds($DurationSeconds)
$startTime = Get-Date
$runspaces = @()

for ($i = 0; $i -lt $ConcurrentSessions; $i++) {
    $ps = [powershell]::Create()
    $ps.RunspacePool = $pool
    [void]$ps.AddScript($scriptBlock).
        AddArgument($TargetUrl).
        AddArgument($TimeoutSec).
        AddArgument($NoCache.IsPresent).
        AddArgument($results).
        AddArgument($Mode).
        AddArgument($TotalRequests).
        AddArgument($endTime)
    $handle = $ps.BeginInvoke()
    $runspaces += [pscustomobject]@{ PS = $ps; Handle = $handle }
}

# --- Suivi en direct pendant l'exécution ---
while (($runspaces | Where-Object { -not $_.Handle.IsCompleted }).Count -gt 0) {
    Start-Sleep -Seconds 2
    $snapshot = $results.ToArray()
    $count = $snapshot.Count
    $okCount = ($snapshot | Where-Object Success).Count
    $koCount = $count - $okCount
    $elapsed = ((Get-Date) - $startTime).TotalSeconds
    $rps = if ($elapsed -gt 0) { [math]::Round($count / $elapsed, 1) } else { 0 }
    Write-Host ("[{0}] Envoyées: {1}  OK={2}  KO={3}  Débit={4} req/s" -f (Get-Date -Format "HH:mm:ss"), $count, $okCount, $koCount, $rps)
}

foreach ($r in $runspaces) {
    $r.PS.EndInvoke($r.Handle) | Out-Null
    $r.PS.Dispose()
}
$pool.Close()
$pool.Dispose()

$duration = (Get-Date) - $startTime
$allResults = $results.ToArray()

# --- Calcul des percentiles ---
function Get-Percentile {
    param([double[]]$Values, [double]$Percentile)
    if ($Values.Count -eq 0) { return 0 }
    $sorted = $Values | Sort-Object
    $idx = [math]::Ceiling($Percentile / 100 * $sorted.Count) - 1
    $idx = [math]::Max(0, [math]::Min($idx, $sorted.Count - 1))
    return $sorted[$idx]
}

$okResults = $allResults | Where-Object Success
$koResults = $allResults | Where-Object { -not $_.Success }
$durations = @($okResults | ForEach-Object { $_.DurationMs })

$avg = if ($durations.Count -gt 0) { ($durations | Measure-Object -Average).Average } else { 0 }
$min = if ($durations.Count -gt 0) { ($durations | Measure-Object -Minimum).Minimum } else { 0 }
$max = if ($durations.Count -gt 0) { ($durations | Measure-Object -Maximum).Maximum } else { 0 }
$p50 = Get-Percentile -Values $durations -Percentile 50
$p90 = Get-Percentile -Values $durations -Percentile 90
$p95 = Get-Percentile -Values $durations -Percentile 95
$p99 = Get-Percentile -Values $durations -Percentile 99

$throughput = if ($duration.TotalSeconds -gt 0) { [math]::Round($allResults.Count / $duration.TotalSeconds, 2) } else { 0 }
$pctKo = if ($allResults.Count -gt 0) { [math]::Round(($koResults.Count / $allResults.Count) * 100, 2) } else { 0 }

Write-Host ""
Write-Host "=== Résultats : $Label ===" -ForegroundColor Cyan
Write-Host "Durée totale        : $([math]::Round($duration.TotalSeconds,2)) s"
Write-Host "Requêtes totales    : $($allResults.Count)"
Write-Host "Succès              : $($okResults.Count)"
Write-Host "Échecs              : $($koResults.Count) ($pctKo %)"
Write-Host "Débit               : $throughput req/s"
Write-Host "Latence moyenne     : $([math]::Round($avg,2)) ms"
Write-Host "Latence min         : $([math]::Round($min,2)) ms"
Write-Host "Latence p50         : $([math]::Round($p50,2)) ms"
Write-Host "Latence p90         : $([math]::Round($p90,2)) ms"
Write-Host "Latence p95         : $([math]::Round($p95,2)) ms"
Write-Host "Latence p99         : $([math]::Round($p99,2)) ms"
Write-Host "Latence max         : $([math]::Round($max,2)) ms"

if ($koResults.Count -gt 0) {
    Write-Host ""
    Write-Host "Répartition des erreurs :" -ForegroundColor Yellow
    $koResults | Group-Object ErrorMsg | Sort-Object Count -Descending | Select-Object -First 5 | ForEach-Object {
        Write-Host ("  {0,5} x  {1}" -f $_.Count, $_.Name)
    }
}

$servedByStats = $okResults | Where-Object { $_.ServedBy } | Group-Object ServedBy
if ($servedByStats) {
    Write-Host ""
    Write-Host "Répartition par backend (en-tête X-Served-By) :" -ForegroundColor Yellow
    $servedByStats | Sort-Object Count -Descending | ForEach-Object {
        Write-Host ("  {0,5} x  {1}" -f $_.Count, $_.Name)
    }
}

$allResults | Sort-Object Timestamp | Export-Csv -Path $ResultFile -NoTypeInformation -Encoding UTF8
Write-Host ""
Write-Host "Résultats exportés dans $ResultFile"

```
## 3. Exécutez le script pour le test sur un seul serveur en 172.17.2.230
```powershell
.\TestDeChargeGroupe.ps1 -TargetUrl "http://172.17.2.230" -Label "Apache-Direct" -ConcurrentSessions 50 -DurationSeconds 60
```
## 4. Exécutez le script pour le test sur 4 serveurs web en backend derrière un load balancer Haproxy en 172.17.2.235
```powershell
.\TestDeChargeGroupe.ps1 -TargetUrl "http://172.17.2.235" -Label "HAProxy-4backends" -ConcurrentSessions 50 -DurationSeconds 60
```
