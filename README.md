# vue.test
# 1. Arrêter le service
Stop-Service -Name "postgresql-x64-17" -Force

# 2. Localiser pg_hba.conf (chemins possibles)
$possiblePaths = @(
    "C:\Program Files\PostgreSQL\17\data\pg_hba.conf",
    "C:\PostgreSQL\17\data\pg_hba.conf",
    "C:\PostgreSQL\data\pg_hba.conf"
)

# Trouver le bon chemin
$pgHbaPath = $possiblePaths | Where-Object { Test-Path $_ } | Select-Object -First 1

if (-not $pgHbaPath) {
    # Rechercher via le registre
    $regPath = Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\postgresql-x64-17" -Name "ImagePath" -ErrorAction SilentlyContinue
    if ($regPath) {
        $match = [regex]::Match($regPath.ImagePath, '-D\s+"([^"]+)"')
        if ($match.Success) {
            $dataDir = $match.Groups[1].Value
            $pgHbaPath = Join-Path $dataDir "pg_hba.conf"
        }
    }
}

if (-not (Test-Path $pgHbaPath)) {
    Write-Host "pg_hba.conf introuvable. Recherche dans tout le système..."
    $pgHbaPath = Get-ChildItem -Path "C:\" -Recurse -Filter "pg_hba.conf" -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName
}

Write-Host "Fichier trouvé : $pgHbaPath"

# 3. Faire une copie de sécurité
Copy-Item $pgHbaPath "$pgHbaPath.backup_$(Get-Date -Format 'yyyyMMdd_HHmmss')"

# 4. Modifier l'authentification (remplacer md5 par trust)
$content = Get-Content $pgHbaPath -Encoding UTF8
$newContent = $content -replace '^\s*host\s+all\s+all\s+127\.0\.0\.1/32\s+md5\s*$', 'host    all             all             127.0.0.1/32            trust'
$newContent = $newContent -replace '^\s*host\s+all\s+all\s+::1/128\s+md5\s*$', 'host    all             all             ::1/128                 trust'
Set-Content $pgHbaPath $newContent -Encoding UTF8
