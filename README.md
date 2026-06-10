# DemoQA Form Automation

UiPath RPA bot that batch-submits student records from Excel to the [DemoQA Practice Form](https://demoqa.com/automation-practice-form), validates each row, and writes a timestamped result report.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Configuration](#configuration)
- [Input File Format](#input-file-format)
- [How to Run](#how-to-run)
- [Output](#output)
- [Design Notes](#design-notes)
- [Assumptions & Known Limitations](#assumptions--known-limitations)
- [Version](#version)

---

## Prerequisites

| Item | Version |
|---|---|
| UiPath Studio | 25.10.9.0 (Windows target) |
| UiPath.UIAutomation.Activities | 24.10.19 |
| UiPath.Excel.Activities | 2.24.4 |
| UiPath.System.Activities | 26.2.6 |
| Google Chrome + UiPath Extension | Latest |

> Requires an unlocked Windows desktop session. Chrome extension must be enabled at `chrome://extensions`.

---

## Project Structure

```
DemoQa_Form_Automation/
│
├── Main.xaml                          # Entry point: orchestrates all phases
│
├── Config/
│   └── Config.xlsx                    # Runtime configuration (paths, URL, retry count)
│
├── Data/
│   ├── Input/
│   │   └── DemoQA_Input.xlsx          # Input records to process
│   ├── Output/                        # Generated output reports (auto-created if not exists)
│   ├── Template/
│   │   └── DemoQA_Output_Template.xlsx # Output file template
│   └── Screenshots/                   # Error screenshots (auto-created if not exists)
│
└── Process/
    ├── Initialization/
    │   ├── InitSettings.xaml          # Loads Config.xlsx into a Dictionary
    │   ├── CheckFileAndFolder.xaml    # Validates paths and loads input data
    │   └── KillAllProcesses.xaml      # Kills residual Excel / Chrome processes
    ├── Logic/
    │   ├── SubmitDemoQA_Form.xaml     # Main form-filling and submission loop
    │   └── ValidateInputData.xaml     # Per-row field validation
    └── Utilities/
        └── TakeScreenshot.xaml        # Captures error screenshots on failure
```

---

## Configuration

`Config/Config.xlsx` — sheet `Settings` (Name / Value columns):

| Name | Default Value |
|---|---|
| `DemoQA_Url` | `https://demoqa.com/automation-practice-form` |
| `InputFilePath` | `Data\Input\DemoQA_Input.xlsx` |
| `InputSheetName` | `Data` |
| `OutputFolderPath` | `Data\Output\` |
| `OutputFileName` | `DemoQA_Output_{DateTime}.xlsx` |
| `OutputSheetName` | `Result` |
| `OutputTemplateFilePath` | `Data\Template\DemoQA_Output_Template.xlsx` |
| `ScreenshotsFolderPath` | `Data\Screenshots\` |
| `RetryCountAttempts` | `3` |

`{DateTime}` is replaced at runtime with `ddMMyyyy_HHmmss`.

---

## Input File Format

File: `Data/Input/DemoQA_Input.xlsx` — Sheet: `Data`

| Column | Type | Required | Validation Rule |
|---|---|---|---|
| `FirstName` | Text | Yes | Must not be empty |
| `LastName` | Text | Yes | Must not be empty |
| `Email` | Text | Yes | Must match `x@x.x` format |
| `Gender` | Text | Yes | Must not be empty (`Male` / `Female` / `Other`) |
| `Mobile` | Text | Yes | Must be exactly 10 digits |
| `DateOfBirth` | Date | Yes | Must not be empty; formatted as `dd MMM yyyy` for the form |
| `Subjects` | Text | Optional | Comma-separated values e.g. `Maths,English` |
| `Hobbies` | Text | Optional | Comma-separated values e.g. `Sports,Reading,Music` |
| `Address` | Text | Optional | Free text |

Rows failing any required-field check are marked `Invalid` and skipped — no browser interaction occurs for them.

---

## How to Run

1. Open `project.json` in UiPath Studio — dependencies resolve automatically.
2. Populate `Data/Input/DemoQA_Input.xlsx` with records.
3. Confirm paths in `Config/Config.xlsx` are correct relative to the project root.
4. Press **F5** (Debug) or **Ctrl+F6** (Run).

The robot kills residual Excel/Chrome processes, loads config, validates files, then iterates each row — validating, filling, and submitting the form.

---

## Output

Output file: `Data/Output/DemoQA_Output_<ddMMyyyy_HHmmss>.xlsx` — sheet: `Result`

All input columns are preserved, plus:

| Column | Description |
|---|---|
| `Status` | `Success` / `Invalid` / `Failed` |
| `Message` | Confirmation text, validation errors, or failure reason + screenshot filename |
| `SubmittedAt` | Submission timestamp (`yyyy-MM-dd HH:mm:ss`); blank if not submitted |

---

## Design Notes

**Selector strategy:** Modern Experience selectors using semantic attributes (`name`, `role`, `check:text`) with fuzzy fallback and anchor-based targeting — resilient to minor DOM changes.

**Validation:** All field errors are collected into a message list before any browser interaction. Rows with any invalid field are skipped entirely and marked `Invalid`.

**Retry & recovery:** Each submission runs inside a `RetryScope`. Before each attempt, the bot checks the page state and refreshes if the form is not detected. After every row (pass or fail), the browser is refreshed to reset form state.

**Error handling:** Exceptions are caught per-row; a screenshot is saved to `Data/Screenshots/` and the filename is appended to the failure message. System-level errors are caught in `Main.xaml` with separate handlers for `Exception` and `BusinessRuleException`.

---

## Assumptions & Known Limitations

- **Desktop session required** — Hardware Events typing does not work headlessly or on a locked screen.
- **Chrome only** — Browser support is currently limited to Google Chrome.
- **Site dependency** — `demoqa.com` is a public third-party site; downtime or DOM changes may cause failures.
- **Subjects must match form options** — Auto-complete requires exact values (e.g. `Maths`, `Biology`).
- **DateOfBirth must be Excel Date type** — Plain text dates are not supported.

---

## Version

**v1.0.1**
