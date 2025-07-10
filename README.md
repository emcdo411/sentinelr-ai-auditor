🛡️ SentinelR: AI-Enhanced Windows System Auditor
A cross-functional project that combines PowerShell, R, and RShiny to scan, analyze, and visualize real-time Windows system performance and security health — built to be usable by both technical and non-technical users.
📌 Table of Contents

📖 Summary
🚀 Features
📦 Tech Stack
📚 Installed Packages
🧠 Why This Matters
🔧 PowerShell Audit Script
🧪 RShiny Dashboard Code
📁 File Structure
📝 UTF-8 Compatibility Fix
✅ Conclusion
🧭 Forking Workflow Diagram

📖 Summary
SentinelR is a mini platform that performs a real-time audit of your Windows system using PowerShell, transforms that data into structured JSON, then renders it in an interactive RShiny dashboard named SysCoach.
It’s built to help:

🧑‍💻 New developers learn real-world scripting, parsing, and UI-building
🧠 Data scientists visualize local system health without needing sysadmin tools
📊 Non-technical stakeholders gain insight into machine status

🚀 Features
✔️ Automated system scan with PowerShell✔️ Clean JSON output for interoperability✔️ Interactive web dashboard via RShiny✔️ Visual tabs for CPU, Services, Defender✔️ PowerBI-style dark UI with Bootstrap 5✔️ AI-powered query responses for system insights  
📦 Tech Stack

  
    
    PowerShellUsed to scan the system and export structured audit reports.
  
  
    
    RProcesses, filters, and transforms the JSON report into visual summaries.
  
  
    
    RShinyProvides a real-time web-based dashboard interface for users.
  
  
    
    JSONUsed as the universal format to move data between PowerShell and R.
  


📚 Installed Packages

  
    
    jsonliteReads and parses PowerShell's structured JSON output.
  
  
    
    DTEnhances RShiny with filterable, paginated data tables.
  
  
    
    bslibAdds dark-mode Bootstrap 5 themes for a modern look.
  
  
    
    shinyFramework for building interactive web applications in RStudio.
  


🧠 Why This Matters
Today’s systems are complex. It’s not enough to just run diagnostic commands — you need visibility, automation, and smart insights. This project bridges the gap between:

🔧 System scripting
📊 Data visualization
🤖 AI-readiness

With SentinelR:

IT admins can audit performance and security without 3rd-party tools
Business users can see what’s running under the hood
Developers and analysts can expand it with AI (e.g., GPT alerts or auto-fixes)

This project can scale into a DevOps monitoring tool, security scanner, or intelligent assistant.
🔧 PowerShell Audit Script
Save as: powershell/sentinel-audit.ps1
$homeFolder = [Environment]::GetFolderPath("MyDocuments")
$projectRoot = Join-Path $homeFolder "SysCoach"
$outputPath = Join-Path $projectRoot "output"

if (!(Test-Path $outputPath)) {
    New-Item -Path $outputPath -ItemType Directory -Force | Out-Null
}

$report = @{}

$report["HighCPUProcesses"] = Get-Process |
    Where-Object { $_.CPU -gt 200 } |
    Select-Object Name, Id, CPU

$report["Services"] = Get-Service |
    Select-Object Name, Status, StartType

try {
    $defender = Get-MpComputerStatus
    $report["Defender"] = $defender | Select-Object AMServiceEnabled, RealTimeProtectionEnabled, AntispywareEnabled
} catch {
    $report["Defender"] = @{ Error = "Defender data not available on this system." }
}

$timestamp = Get-Date -Format "yyyyMMdd_HHmmss"
$filename = "sys_report_$timestamp.json"
$outputFile = Join-Path $outputPath $filename

# ✅ UTF-8 Safe Write
$json = $report | ConvertTo-Json -Depth 5
[System.IO.File]::WriteAllText($outputFile, $json, [System.Text.Encoding]::UTF8)

Write-Output "✅ Audit complete. File saved to:"
Write-Output $outputFile

🧪 RShiny Dashboard Code
Save as: R/app.R
library(shiny)
library(jsonlite)
library(DT)
library(bslib)

# 🔹 Helper: Load latest sys_report JSON + filename
get_latest_report <- function() {
  search_paths <- c(
    "C:/Users/Veteran/Documents/SysCoach/output",
    "output",
    "../output",
    file.path(getwd(), "output")
  )

  for (path in search_paths) {
    files <- list.files(path, pattern = "sys_report_.*\\.json$|sentinel_report_.*\\.json$", full.names = TRUE)
    if (length(files) > 0) {
      latest <- files[which.max(file.info(files)$mtime)]
      return(list(
        file = basename(latest),
        data = tryCatch(jsonlite::fromJSON(latest), error = function(e) NULL)
      ))
    }
  }

  return(list(
    file = "❌ No report found",
    data = NULL
  ))
}

# 🔹 Theme
dark_theme <- bs_theme(
  version = 5,
  bootswatch = "darkly",
  base_font = font_google("Roboto")
)

# 🔹 UI
ui <- fluidPage(
  theme = dark_theme,
  titlePanel("🧠 SysCoach - AI-Powered System Companion"),
  sidebarLayout(
    sidebarPanel(
      textInput("query", "Ask SysCoach:", placeholder = "e.g., What's using the most memory?"),
      actionButton("submit", "🔍 Analyze"),
      br(), br(),
      strong("📁 Report Loaded:"),
      textOutput("report_name"),
      br(),
      verbatimTextOutput("ai_response"),
      width = 4
    ),
    mainPanel(
      tabsetPanel(
        tabPanel("🔥 High CPU", DTOutput("cpu")),
        tabPanel("🔧 Services", DTOutput("services")),
        tabPanel("🛡️ Defender", DTOutput("defender"))
      ),
      width = 8
    )
  )
)

# 🔹 Server
server <- function(input, output, session) {
  report_data <- reactiveVal()
  
  observeEvent(input$submit, {
    report <- get_latest_report()
    report_data(report)
  })
  
  observe({
    report <- get_latest_report()
    report_data(report)
  })
  
  output$report_name <- renderText({
    req(report_data()$file)
    report_data()$file
  })
  
  output$ai_response <- renderText({
    req(input$submit)
    sysdata <- report_data()$data
    if (is.null(sysdata)) return("⚠️ No system data available.")
    
    q <- tolower(input$query)
    
    if (grepl("cpu|slow|high usage", q)) {
      if (!is.data.frame(sysdata$HighCPUProcesses) || !"CPU" %in% names(sysdata$HighCPUProcesses)) {
        return("⚠️ No valid CPU data found.")
      }
      cpu <- sysdata$HighCPUProcesses
      cpu$CPU <- suppressWarnings(as.numeric(cpu$CPU))
      cpu <- cpu[!is.na(cpu$CPU), ]
      if (nrow(cpu) == 0) {
        return("⚠️ No valid CPU data available.")
      }
      top_cpu <- cpu[order(-cpu$CPU), ][1:min(3, nrow(cpu)), ]
      paste0("Top CPU processes:\n", 
             paste(top_cpu$Name, round(top_cpu$CPU, 2), sep = " - ", collapse = "\n"))
      
    } else if (grepl("services|stopped", q)) {
      if (!is.data.frame(sysdata$Services) || !"Status" %in% names(sysdata$Services)) {
        return("⚠️ No valid services data found.")
      }
      svc <- sysdata$Services
      svc$StatusText <- ifelse(svc$Status == "Running", "Running", "Stopped")
      stopped <- svc[svc$StatusText != "Running", ]
      if (nrow(stopped) == 0) {
        return("✅ All services are running or no stopped services found.")
      }
      paste0("Stopped services (top 5):\n", 
             paste(head(stopped$Name, 5), collapse = ", "))
      
    } else if (grepl("defender|antivirus", q)) {
      if (!is.list(sysdata$Defender)) {
        return("⚠️ No Defender data found.")
      }
      def <- as.data.frame(sysdata$Defender)
      if (nrow(def) == 0) {
        return("⚠️ No Defender data available.")
      }
      paste0("Defender Status:\n", 
             paste(names(def), def[1, ], sep = ": ", collapse = "\n"))
      
    } else {
      paste("🧠 Sorry, I don't understand that yet.\nYou asked:", input$query)
    }
  })
  
  output$cpu <- renderDT({
    req(report_data()$data)
    if (!is.data.frame(report_data()$data$HighCPUProcesses)) {
      datatable(data.frame(Message = "No CPU data available"))
    } else {
      datatable(report_data()$data$HighCPUProcesses)
    }
  })
  
  output$services <- renderDT({
    req(report_data()$data)
    if (!is.data.frame(report_data()$data$Services)) {
      datatable(data.frame(Message = "No services data available"))
    } else {
      datatable(report_data()$data$Services)
    }
  })
  
  output$defender <- renderDT({
    req(report_data()$data)
    if (!is.list(report_data()$data$Defender)) {
      datatable(data.frame(Message = "No Defender data available"))
    } else {
      datatable(as.data.frame(report_data()$data$Defender))
    }
  })
}

# 🔹 Launch app
shinyApp(ui = ui, server = server)

📁 File Structure
sentinelr-ai-auditor/
├── powershell/
│   └── sentinel-audit.ps1
├── R/
│   └── app.R
├── output/
│   └── sys_report_<timestamp>.json
├── README.md

📝 UTF-8 Compatibility Fix
PowerShell's default file output uses UTF-16 encoding, which causes errors when loading data into R. To resolve this, the script explicitly writes files using UTF-8 encoding:
[System.IO.File]::WriteAllText($outputFile, $json, [System.Text.Encoding]::UTF8)

This ensures compatibility with R’s jsonlite::fromJSON() and avoids parsing failures.
✅ Conclusion
This project combines real-world scripting, structured data handling, and modern visualization — all in one place. Whether you're learning, troubleshooting, or building, SentinelR gives you a scalable, smart way to interact with system diagnostics — and it's built to evolve with AI capabilities.

Try it. Fork it. Expand it. Make your own AI-driven system assistant.

🧭 Forking Workflow Diagram
flowchart LR
    A[Original Repo: main] -->|Fork| B[Your Fork: main]
    B -->|Create Branch| C[feature/login]
    C -->|Commit| D[Commits]
    D -->|Push to Fork| E[Pull Request]
    E -->|Merge| F[Original Repo: main]

