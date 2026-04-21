# AI-PROJECT
AI PROJECT

# Agentic Data Pipeline System

A multi-agent system for Excel data normalization that replaces agent names with business names, splits data by status codes (A/C/P), and writes formatted output into Excel templates.

## System Overview

This system uses three coordinating agents to process exported data:
- **Orchestrator Agent**: Routes data and manages the pipeline
- **Transformer Agent**: Replaces agent names with business names (LLM-powered with FLAN-T5 fallback)
- **Writer Agent**: Splits data by status and writes to Excel template

## Setup Instructions

### Prerequisites

- Python 3.9 or higher
- pip package manager

### Installation

1. Clone the repository:
```bash
git clone <your-repo-url>
cd <repo-name>

Install dependencies:

bash
pip install pandas openpyxl pydantic
(Optional) For LLM-powered transformation:

bash
pip install transformers torch
Required Input Files
You need three input files:

Template Excel file (.xlsx) - A blank Excel template with formatting

Export data file (.xlsx, .xltx, or .csv) - Contains the data to process

Agents file (.xlsx or .csv) - Must have columns: agent_name, business_name

Running the System
Basic usage:

bashpython
With options:

bash
python pipeline.py \
  --template template.xlsx \
  --export data.csv \
  --agents agents.xlsx \
  --output output.xlsx \
  --sort-by agent_name \
  --drop-columns unused_col1 unused_col2 \
  --money-columns salary budget \
  --start-row 10 \
  --gap-rows 4
Command Line Arguments
Argument	Required	Description
--template	Yes	Path to blank template .xlsx
--export	Yes	Path to exported data file
--agents	Yes	Path to agents mapping file
--output	Yes	Path for output .xlsx
--sort-by	No	Column name to sort by
--drop-columns	No	Columns to remove from output
--money-columns	No	Columns to format as currency
--model-name	No	HuggingFace model (default: google/flan-t5-small)
--start-row	No	Row number to start writing (default: 10)
--gap-rows	No	Blank rows between sections (default: 4)
Expected Output
The system produces an Excel file with:

Data split into three sections: A, C, P (based on status column)

Agent names replaced with business names (or bolded if no mapping)

Money columns formatted as $X,XXX.XX

Sorted and filtered according to your configuration

Troubleshooting
"Could not auto-detect status column" - Your export data must contain a column with only values A, C, or P.

Transformers not installed warning - System falls back to deterministic replacement. Install transformers for LLM mode.

Invalid agents row - Check that agent_name is not empty and business_name is valid text.
