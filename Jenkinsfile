pipeline {
  agent { label 'windows' }

  options { timestamps(); ansiColor('xterm') }

  parameters {
    choice(name: 'ENVIRONMENT', choices: ['Dev', 'Prod'], description: 'Select Environment')
  }

  environment {
    PROJECT_NAME = 'Demo Report'
    BASE_FOLDER  = "${WORKSPACE}\\PbipReports"
    CONFIG_FILE  = "${WORKSPACE}\\config.json"
  }

  triggers {
    // Use a GitHub webhook for pushes. Leave polling as a fallback if you can't expose Jenkins.
    githubPush()
    // pollSCM('H/10 * * * *')  // optional fallback
  }

  stages {
    stage('Checkout') {
      steps {
        // If the job is "Pipeline script from SCM", Jenkins already fetched to read Jenkinsfile,
        // but this ensures the workspace has the repo content.
        checkout scm
      }
    }

    stage('Prep PowerShell') {
      steps {
        powershell(script: '''
          $ErrorActionPreference = "Stop"

          # Avoid TLS issues when hitting GitHub/PSGallery on older hosts:
          try { [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12 } catch {}

          # Trust PSGallery (skip prompts)
          try { Set-PSRepository -Name "PSGallery" -InstallationPolicy Trusted -ErrorAction SilentlyContinue } catch {}

          # Ensure NuGet provider is available (skip confirmation)
          if (-not (Get-PackageProvider -Name NuGet -ErrorAction SilentlyContinue)) {
            Install-PackageProvider -Name NuGet -MinimumVersion 2.8.5.201 -Force -Scope CurrentUser
          }

          # Install Az.Accounts silently if missing
          if (-not (Get-Module -ListAvailable -Name Az.Accounts)) {
            Install-Module Az.Accounts -Scope CurrentUser -Force -AllowClobber -Confirm:$false
          }
        ''')
      }
    }


    stage('Publish PBIP to Fabric') {
      steps {
        powershell(script: '''
          $ErrorActionPreference = "Stop"

          # -------- Load config + pick environment --------
          $configPath = "$env:CONFIG_FILE"
          if (!(Test-Path $configPath)) { Write-Error "❌ Config file not found at $configPath"; exit 1 }
          $config = Get-Content $configPath | ConvertFrom-Json

          $environment = "${env:ENVIRONMENT}"
          Write-Host "🌍 Selected Environment: $environment"

          if ($environment -eq "Dev") {
            $WorkspaceId   = $config.DevWorkspaceID
            $WorkspaceName = "Dev"
            $ServerName    = $config.DevWarehouseConnection
            $DatabaseName  = $config.DevWarehouseName
          } elseif ($environment -eq "Prod") {
            $WorkspaceId   = $config.ProdWorkspaceID
            $WorkspaceName = "Prod"
            $ServerName    = $config.ProdWarehouseConnection
            $DatabaseName  = $config.ProdWarehouseName
          } else {
            Write-Error "❌ Invalid environment selected: $environment"; exit 1
          }

          if ([string]::IsNullOrWhiteSpace($ServerName)) { Write-Error "❌ Server name is empty for $environment"; exit 1 }
          if ([string]::IsNullOrWhiteSpace($DatabaseName)) { Write-Error "❌ Database name is empty for $environment"; exit 1 }

          # -------- Common values --------
          $TenantId     = $config.TenantID
          $ClientId     = $config.ClientID
          $ClientSecret = $config.ClientSecret
          $projectName  = "$env:PROJECT_NAME"
          $pbipFolder   = "$env:BASE_FOLDER"

          Write-Host "🔐 Authenticating to Fabric..."
          $moduleFolder = Join-Path $env:WORKSPACE "module"
          New-Item -ItemType Directory -Path $moduleFolder -ErrorAction SilentlyContinue | Out-Null

          @(
            "https://raw.githubusercontent.com/microsoft/Analysis-Services/master/pbidevmode/fabricps-pbip/FabricPS-PBIP.psm1",
            "https://raw.githubusercontent.com/microsoft/Analysis-Services/master/pbidevmode/fabricps-pbip/FabricPS-PBIP.psd1"
          ) | ForEach-Object {
            $file = Split-Path $_ -Leaf
            Invoke-WebRequest -Uri $_ -OutFile (Join-Path $moduleFolder $file) -UseBasicParsing
          }

          Import-Module Az.Accounts -Force
          Import-Module (Join-Path $moduleFolder "FabricPS-PBIP.psd1") -Force

          # Fabric Authentication (Service Principal)
          Set-FabricAuthToken `
            -servicePrincipalId $ClientId `
            -servicePrincipalSecret $ClientSecret `
            -tenantId $TenantId `
            -reset

          # -------- Paths --------
          $semanticModelPath = Join-Path $pbipFolder "$projectName.SemanticModel"
          $reportPath        = Join-Path $pbipFolder "$projectName.Report"

          if (!(Test-Path $semanticModelPath) -or !(Test-Path $reportPath)) {
            Write-Error "❌ Missing .SemanticModel or .Report folder for '$projectName' in $pbipFolder"; exit 1
          }

          # -------- Switch connection strings in model.bim --------
          try {
            Write-Host "🔄 Updating semantic model data source connection..."
            $modelPath = Join-Path $semanticModelPath "model.bim"
            if (!(Test-Path $modelPath)) { throw "model.bim not found at $modelPath" }

            $modelJson = (Get-Content -Path $modelPath -Raw) | ConvertFrom-Json
            $sqlPattern = 'Sql\\.Database\\s*\\(\\s*"[^"]*"\\s*,\\s*"[^"]*"[^)]*\\)'
            $replacement = 'Sql.Database("' + $ServerName + '", "' + $DatabaseName + '")'
            $updatesApplied = 0

            foreach ($table in $modelJson.model.tables) {
              if ($table.partitions) {
                foreach ($partition in $table.partitions) {
                  if ($partition.source -and $partition.source.type -eq 'm' -and $partition.source.expression) {
                    if ($partition.source.expression -is [System.Array]) {
                      $newExpression = @()
                      $hasChanges = $false
                      foreach ($line in $partition.source.expression) {
                        if ($line -match $sqlPattern) {
                          $newExpression += ($line -replace $sqlPattern, $replacement)
                          $hasChanges = $true
                          Write-Host "📝 Updated '$($table.name)' / '$($partition.name)'"
                        } else { $newExpression += $line }
                      }
                      if ($hasChanges) { $partition.source.expression = $newExpression; $updatesApplied++ }
                    } elseif ($partition.source.expression -is [string]) {
                      if ($partition.source.expression -match $sqlPattern) {
                        $partition.source.expression = $partition.source.expression -replace $sqlPattern, $replacement
                        $updatesApplied++
                        Write-Host "📝 Updated '$($table.name)' / '$($partition.name)'"
                      }
                    }
                  }
                }
              }
            }

            if ($updatesApplied -gt 0) {
              $modelJson | ConvertTo-Json -Depth 100 | Set-Content -Path $modelPath -Encoding UTF8
              Write-Host "✅ Connection switched in $updatesApplied partition(s) to $DatabaseName ($ServerName)"
            } else {
              Write-Host "⚠️ No SQL Database connections found to update."
            }
          } catch {
            Write-Error "Failed to update semantic model connection: $_"; exit 1
          }

          # -------- Import Semantic Model --------
          Write-Host "📦 Importing Semantic Model..."
          try {
            $semanticModelImport = Import-FabricItem -workspaceId $WorkspaceId -path $semanticModelPath -ErrorAction Stop
            Write-Host "✅ Semantic model import successful. ID: $($semanticModelImport.Id)"
          } catch {
            Write-Error "❌ Semantic model import failed: $($_.Exception.Message)"; exit 1
          }

          # -------- Verify Datasource --------
          Write-Host "🔐 Verifying data source configuration..."
          try {
            $token = Get-FabricAuthToken
            $headers = @{ "Authorization" = "Bearer $token"; "Content-Type" = "application/json" }
            $datasourcesUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$($semanticModelImport.Id)/datasources"
            $datasources = Invoke-RestMethod -Uri $datasourcesUrl -Headers $headers -Method Get -ErrorAction Stop
            
            if ($datasources -and $datasources.value -and $datasources.value.Count -gt 0) {
              Write-Host "✅ Found $($datasources.value.Count) datasource(s)"
              foreach ($ds in $datasources.value) {
                Write-Host "  - Type: $($ds.datasourceType)"
                Write-Host "    Server: $($ds.connectionDetails.server)"
                Write-Host "    Database: $($ds.connectionDetails.database)"
                if (![string]::IsNullOrWhiteSpace($ds.datasourceId)) {
                  Write-Host "    DatasourceId: $($ds.datasourceId)"
                } else {
                  Write-Host "    DatasourceId: [Empty - normal for some Fabric connections]"
                }
              }
              Start-Sleep -Seconds 10
            } else {
              Write-Host "⚠️ No datasources found."
            }
          } catch {
            Write-Host "⚠️ Could not verify datasource configuration: $($_.Exception.Message)"
          }

          # -------- Import Report + bind --------
          Write-Host "📊 Importing Report and binding to Semantic Model..."
          try {
            $reportImport = Import-FabricItem -workspaceId $WorkspaceId -path $reportPath -itemProperties @{ "semanticModelId" = $semanticModelImport.Id } -ErrorAction Stop
            Write-Host "✅ Report imported and bound. ID: $($reportImport.Id)"
          } catch {
            Write-Error "❌ Failed to import Report: $_"; exit 1
          }

          # -------- Optional: test refresh --------
          Write-Host "🧪 Testing data connection..."
          try {
            $refreshUrl = "https://api.powerbi.com/v1.0/myorg/groups/$WorkspaceId/datasets/$($semanticModelImport.Id)/refreshes"
            $refreshPayload = @{ type = "full"; commitMode = "transactional"; maxParallelism = 2; retryCount = 2 } | ConvertTo-Json
            $refreshResponse = Invoke-RestMethod -Uri $refreshUrl -Headers $headers -Method Post -Body $refreshPayload -ErrorAction Stop
            Write-Host "✅ Data refresh initiated. Refresh ID: $($refreshResponse.requestId)"
          } catch {
            Write-Host "ℹ️ Could not initiate test refresh: $($_.Exception.Message)"
          }

          # -------- Summary --------
          Write-Host ""
          Write-Host "🎉 Successfully published '$projectName' to workspace '$WorkspaceName'"
          Write-Host "ℹ️ Warehouse: $DatabaseName | Connection: $ServerName"
          Write-Host "🌐 Links:"
          Write-Host "  📊 Report: https://app.fabric.microsoft.com/groups/$WorkspaceId/reports/$($reportImport.Id)"
          Write-Host "  📈 Semantic Model: https://app.fabric.microsoft.com/groups/$WorkspaceId/semanticmodels/$($semanticModelImport.Id)"
        ''')
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'module/**', onlyIfSuccessful: false
    }
  }
}
