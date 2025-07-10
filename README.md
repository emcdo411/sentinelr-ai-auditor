````markdown
# 🛡️ SentinelR: AI-Enhanced Windows System Auditor

A cross-functional project that combines PowerShell, R, and RShiny to scan, analyze, and visualize real-time system performance and security health — even for non-technical users.

---

## 📌 Table of Contents
- [📖 Summary](#📖-summary)
- [🚀 Features](#🚀-features)
- [📦 Tech Stack](#📦-tech-stack)
- [📚 Installed Packages](#📚-installed-packages)
- [🧠 Why This Matters](#🧠-why-this-matters)
- [🔧 PowerShell Audit Script](#🔧-powershell-audit-script)
- [🧪 RShiny Dashboard Code](#🧪-rshiny-dashboard-code)
- [📁 File Structure](#📁-file-structure)
- [📝 UTF-8 Compatibility Fix](#📝-utf-8-compatibility-fix)
- [✅ Conclusion](#✅-conclusion)

---

## 📖 Summary

**SentinelR** is an intelligent system auditing toolkit designed for both technical and non-technical users. Using PowerShell, it collects important system health data such as high-CPU processes, stopped services, and security status. That data is then parsed and visualized in a live **RShiny dashboard**, which anyone can interact with — no terminal skills required.

Whether you're a:
- 🔰 **New developer** learning how systems work,
- 🧠 **Data scientist** building insights into device health,
- 🧑‍💼 **Analyst or manager** who wants visibility into Windows performance,

This project shows how automation, AI, and visualization can bring complex system data to life.

---

## 🚀 Features

✔️ Scans Windows systems with PowerShell  
✔️ Stores structured results in JSON  
✔️ Loads + analyzes results in R  
✔️ Interactive RShiny dashboard  
✔️ Auto-refreshable interface  
✔️ Styled with dark mode and professional layout  
✔️ Designed for future AI integration (e.g., GPT-assisted remediation)

---

## 📦 Tech Stack

![PowerShell](https://img.shields.io/badge/PowerShell-0078D4?logo=powershell&logoColor=white)
![R](https://img.shields.io/badge/R-276DC3?logo=r&logoColor=white)
![RShiny](https://img.shields.io/badge/Shiny%20R-20232A?logo=r&logoColor=white)
![JSON](https://img.shields.io/badge/JSON-000000?logo=json&logoColor=white)

---

## 📚 Installed Packages

![jsonlite](https://img.shields.io/badge/jsonlite-%20-lightgrey)
![DT](https://img.shields.io/badge/DT-Interactive-lightblue)
![bslib](https://img.shields.io/badge/bslib-Themes-darkgreen)
![shiny](https://img.shields.io/badge/shiny-Web%20App%20Engine-purple)

---

## 🧠 Why This Matters

In today’s AI-powered workplace, it’s not enough to just gather data — you must **make it actionable and human-readable**. SentinelR is a model of how technical and non-technical users can collaborate:

- 🔧 IT Admins get visibility without jumping into Task Manager
- 📊 Analysts can track system health in real-time
- 🤖 Developers can embed AI tools like GPT to flag and fix issues

It’s also a practical stepping stone into:
- DevOps
- Platform engineering
- Cybersecurity automation
- AI-powered troubleshooting

---

## 🔧 PowerShell Audit Script

Save this file as: `sentinel-audit.ps1`

```powershell
# SentinelR - System Audit Script (PowerShell)
$homeFolder = [Environment]::GetFolderPath("MyDocuments")
$projectRoot = Join-Path $homeFolder "SentinelR"
$outputPath = Join-Path $projectRoot "output"

if (!(Test-Path $outputPath)) {
    New-Item -Path $outputPath -ItemType Directory -Force | Out-Null
}

$report = @{}

$report["HighCPUProcesses"] = Get-Process |
    Where-Object { $_.CPU -gt 200 } |
    Select-Object Name, Id, CPU

$report["AutoServicesNotRunning"] = Get-Service |
    Where-Object { $_.StartType -eq 'Automatic' -and $_.Status -ne 'Running' } |
    Select-Object Name, Status

$report["ScheduledTasks"] = Get-ScheduledTask |
    Select-Object TaskName, State

try {
    $defender = Get-MpComputerStatus
    $report["Defender"] = $defender | Select-Object AMServiceEnabled, RealTimeProtectionEnabled, AntispywareEnabled
} catch {
    $report["Defender"] = @{ Error = "Defender data not available on this system." }
}

$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$filename = "sentinel_report_$timestamp.json"
$outputFile = Join-Path $outputPath $filename

# ✅ UTF-8 Safe Write
$json = $report | ConvertTo-Json -Depth 5
[System.IO.File]::WriteAllText($outputFile, $json, [System.Text.Encoding]::UTF8)

Write-Output "✅ Audit complete. File saved to:"
Write-Output $outputFile
````

---

## 🧪 RShiny Dashboard Code

Save this file as: `app.R`

```r
library(shiny)
library(jsonlite)
library(DT)
library(bslib)

get_latest_json <- function() {
  output_dir <- file.path(Sys.getenv("USERPROFILE"), "Documents", "SentinelR", "output")
  json_files <- list.files(output_dir, pattern = "sentinel_report_.*\\.json$", full.names = TRUE)
  if (length(json_files) == 0) return(NULL)
  json_files[which.max(file.info(json_files)$mtime)]
}

dark_theme <- bs_theme(
  version = 5,
  bootswatch = "darkly",
  base_font = font_google("Roboto")
)

ui <- fluidPage(
  theme = dark_theme,
  tags$head(
    tags$style(HTML("
      .title-card {
        background-color: #212529;
        color: #F8F9FA;
        border-left: 6px solid #28a745;
        padding: 15px;
        margin-bottom: 20px;
        border-radius: 10px;
      }
      .file-info {
        color: #6c757d;
        font-size: 0.9em;
        margin-bottom: 15px;
      }
    "))
  ),
  div(class = "title-card",
      h2("🛡️ SentinelR Dashboard"),
      p("AI-Enhanced Windows System Auditor")
  ),
  div(class = "file-info", textOutput("file_path")),
  actionButton("refresh", "🔄 Refresh Data"),
  br(), br(),
  tabsetPanel(
    tabPanel("🧠 Defender Status", DTOutput("defender")),
    tabPanel("🔥 High CPU Processes", DTOutput("high_cpu")),
    tabPanel("🔧 Auto Services Not Running", DTOutput("auto_services")),
    tabPanel("⏰ Scheduled Tasks", DTOutput("tasks"))
  )
)

server <- function(input, output, session) {
  current_data <- reactiveVal()
  current_file <- reactiveVal()

  load_data <- function() {
    file <- get_latest_json()
    if (!is.null(file)) {
      json <- fromJSON(file)
      current_data(json)
      current_file(file)
    }
  }

  observeEvent(input$refresh, load_data, ignoreInit = TRUE)
  load_data()

  output$file_path <- renderText({
    req(current_file())
    paste("Last loaded:", basename(current_file()))
  })

  output$defender <- renderDT({
    req(current_data())
    datatable(as.data.frame(current_data()$Defender), options = list(pageLength = 5))
  })

  output$high_cpu <- renderDT({
    req(current_data())
    datatable(current_data()$HighCPUProcesses, options = list(pageLength = 10))
  })

  output$auto_services <- renderDT({
    req(current_data())
    datatable(current_data()$AutoServicesNotRunning, options = list(pageLength = 10))
  })

  output$tasks <- renderDT({
    req(current_data())
    datatable(current_data()$ScheduledTasks, options = list(pageLength = 10))
  })
}

shinyApp(ui = ui, server = server)
```

---

## 📁 File Structure

```
sentinelr-ai-auditor/
├── powershell/
│   └── sentinel-audit.ps1
├── R/
│   └── app.R
├── output/
│   └── sentinel_report_<timestamp>.json
├── README.md
```

---

## 📝 UTF-8 Compatibility Fix

PowerShell defaults to UTF-16 for text files, which **breaks R’s JSON parser**. To fix this:

✅ This project uses:

```powershell
[System.IO.File]::WriteAllText(..., [System.Text.Encoding]::UTF8)
```

This ensures your JSON files are saved in clean, **UTF-8 encoding**, compatible with `jsonlite::fromJSON()` in R.

---

## ✅ Conclusion

**SentinelR** isn’t just a code project — it’s a bridge between:

* 🔧 system diagnostics
* 📊 interactive reporting
* 🤖 AI-readiness
* 🧠 and explainability for *all levels of users*

Whether you’re debugging a workstation, auditing a client machine, or just learning how systems and AI can work together — **this repo is a powerful starting point.**

---

### ✨ Want to take it further?

* Add GPT to explain or fix issues
* Trigger auto-remediation actions from the Shiny dashboard
* Upload logs to a cloud database for team-wide visibility

---

Let me know if you’d like this converted into a real repo with commit-ready structure, or if you want help adding GPT into the pipeline!

```
```
