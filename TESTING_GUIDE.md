# Testing Guide for Multi-Year PDF Processing

## Quick Start

### 1. Start the Flask App
```bash
cd c:\Users\bkuro\Documents\GitHub\pohagan72-OurFamilyWizard-Message-Analyzer
python app.py
```

You should see:
```
 * Running on http://0.0.0.0:5000
 * Debug mode: on
```

### 2. Open Browser
Navigate to: `http://localhost:5000`

### 3. Upload a Multi-Year PDF
- Click "Choose File" and select a PDF with messages from multiple years
- Click "Upload and Process Messages"
- Wait for the "raw message logs" to upload

You should see: `"Successfully processed PDF and uploaded all X monthly raw message logs"`

### 4. Generate Analysis Reports
- Select AI Model (Gemini or Azure OpenAI)
- Click "Generate Analysis Reports"
- Monitor the server console for detailed logging

## Log Monitoring

Open a **separate terminal** while the app runs to watch logs in real-time:

```bash
# PowerShell
cd c:\Users\bkuro\Documents\GitHub\pohagan72-OurFamilyWizard-Message-Analyzer
python app.py 2>&1 | Tee-Object -FilePath test.log

# Then in another terminal:
Get-Content test.log -Wait
```

## Expected Log Sequence for 2-Year File (2022-2023)

### Upload Phase (1-2 minutes)
```
--- Received New Upload Request ---
Reading PDF file into memory...
Extracted X characters. Participants detected: 'Person A | Person B'
Parsing messages...
Parsed messages into 2 years: [2022, 2023]
Total distinct Year/Month combinations found: 24
Saving parsed message structure to temporary file...
Starting PARALLEL upload of 24 raw monthly message logs...
--- Initial processing & parallel raw log upload complete in XXs ---
```

### Report Generation Phase (10-30 minutes depending on file size)

#### Monthly Processing:
```
======= Starting PARALLEL Monthly Analysis (24 months) using up to 3 workers =======
  Completed 10/24 monthly tasks...
  Completed 20/24 monthly tasks...
Finished PARALLEL Monthly Analysis
```

#### Yearly Processing (THE CRITICAL SECTION):
```
Found X successful monthly analyses across 2 years for yearly aggregation.

======= Starting PARALLEL Yearly Analysis (2 years) using up to 2 workers =======
[YEARLY EXECUTOR] Created with 2 workers, submitting 2 yearly tasks...
[YEARLY EXECUTOR] All 2 yearly tasks submitted. Beginning collection phase...
[YEARLY COLLECTION] Starting to collect 2 yearly task results with 3600s timeout per task...

>>>>> [YEARLY TASK START] Year 2022 - 14:25:30 - Starting monthly data aggregation (12 months)
>>>>> [YEARLY TASK START] Year 2023 - 14:25:31 - Starting monthly data aggregation (12 months)

Year 2022: Calling generate_yearly_report_data_and_pdf() - 14:25:31
Year 2022: PDF generation returned at 14:30:45: /tmp/ofw_xxxxxxxx/yearly_2022.pdf
Year 2022: Uploading yearly report to Azure: ofw_extract_xxx/reports/gemini/2022/yearly_analysis_report_2022.pdf - 14:30:46
Year 2022: Azure upload completed - 14:30:50

<<< [YEARLY TASK SUCCESS] Year 2022 - Elapsed 319.2s - Status: success

Year 2023: Calling generate_yearly_report_data_and_pdf() - 14:25:35
Year 2023: PDF generation returned at 14:31:20: /tmp/ofw_xxxxxxxx/yearly_2023.pdf
Year 2023: Uploading yearly report to Azure: ofw_extract_xxx/reports/gemini/2023/yearly_analysis_report_2023.pdf - 14:31:21
Year 2023: Azure upload completed - 14:31:25

<<< [YEARLY TASK SUCCESS] Year 2023 - Elapsed 354.8s - Status: success

======= Finished PARALLEL Yearly Analysis =======
```

## Expected Outcomes by File Type

### ✅ Small Single-Year File (1 month - 5 months)
**Duration**: 2-5 minutes
**No yearly processing** (only 1 year)
```
Parsed messages into 1 years: [2023]
Total distinct Year/Month combinations found: 5
======= Finished PARALLEL Monthly Analysis =======
```

### ✅ Medium Multi-Year File (2022 + 2023, 2 months - 12 months per year)
**Duration**: 15-45 minutes
```
Parsed messages into 2 years: [2022, 2023]
Total distinct Year/Month combinations found: 24
======= Starting PARALLEL Yearly Analysis (2 years) using up to 2 workers =======
<<< [YEARLY TASK SUCCESS] Year 2022 - Elapsed 280s - Status: success
<<< [YEARLY TASK SUCCESS] Year 2023 - Elapsed 310s - Status: success
```

### ⚠️ Large Multi-Year File (2022 + 2023 + 2024, 12 months each = 36 months)
**Duration**: 45-90 minutes
**Potential Timeout Risk**: If any year takes >3600s
```
======= Starting PARALLEL Yearly Analysis (3 years) using up to 3 workers =======
[YEARLY EXECUTOR] Created with 3 workers, submitting 3 yearly tasks...
[YEARLY COLLECTION] Starting to collect 3 yearly task results with 3600s timeout per task...
```

### 🔴 If Freeze Occurs (Before Fix)
```
[YEARLY EXECUTOR] All 2 yearly tasks submitted. Beginning collection phase...
[YEARLY COLLECTION] Starting to collect 2 yearly task results with 3600s timeout per task...
# HANGS HERE INDEFINITELY
# Output stops, no further progress logged
```

### 🟢 If Freeze Was Happening (After Fix)
```
[YEARLY EXECUTOR] All 2 yearly tasks submitted. Beginning collection phase...
[YEARLY COLLECTION] Starting to collect 2 yearly task results with 3600s timeout per task...
!!! TIMEOUT: Yearly processing for 2024 exceeded 3600s limit
```

## Troubleshooting

### Scenario 1: "Freeze Hangs for 30+ minutes"
**Check logs for**:
- Does `[YEARLY TASK START]` appear for all years?
- Does any year have `<<< [YEARLY TASK SUCCESS]`?
- Are there any timeout messages?

**If No TASK START Messages**:
- Executor may not be creating tasks correctly
- Check that `yearly_tasks` list is populated

**If TASK START but No SUCCESS**:
- Task is running but not completing
- Check if it's hitting the 3600s timeout
- Look for API errors in the logs

### Scenario 2: "PDF Generation Appears Stuck"
**Check logs for**:
```
Year 2023: Calling generate_yearly_report_data_and_pdf() - 14:30:00
Year 2023: PDF generation returned at 14:35:00
```

If the second line never appears after 10+ minutes, the AI analysis or PDF generation is hanging.

**Solutions**:
1. Increase `YEARLY_TASK_TIMEOUT` if needed for very large months
2. Check if Azure OpenAI or Google Gemini API is responding
3. Monitor network connectivity

### Scenario 3: "One Year Succeeds, Second Year Doesn't Appear"
**Likely Cause**: Thread pool issue or memory exhaustion

**Check**:
1. Are there any exception messages after the first year completes?
2. Check system memory usage: `Get-Process python | Format-Table WorkingSet`
3. Check if Azure upload is blocking

**Solution**:
```python
# In app.py, find YEARLY_TASK_TIMEOUT definition
# Reduce concurrent workers:
num_yearly_workers = min(2, len(years_to_process_yearly))  # Use only 2 workers
```

## Manual Testing Commands

### Test PDF Extraction
```python
from app import extract_text_and_participants
import io

with open('your_test.pdf', 'rb') as f:
    pdf_bytes = f.read()

text, participants_str, participants_list = extract_text_and_participants(io.BytesIO(pdf_bytes))
print(f"Extracted {len(text)} characters")
print(f"Participants: {participants_str}")
```

### Test Message Parsing
```python
from app import parse_messages

with open('your_test.pdf', 'rb') as f:
    from app import extract_text_and_participants, clean_message_text, parse_messages
    import io
    text, _, _ = extract_text_and_participants(io.BytesIO(f.read()))
    cleaned = clean_message_text(text)
    messages = parse_messages(cleaned)
    
    for year in sorted(messages.keys()):
        print(f"Year {year}: {len(messages[year])} months")
```

## Cleanup

### Remove Old Test Files
```bash
# Remove temporary data
Remove-Item c:\Users\bkuro\AppData\Local\Temp\ofw_*.json -ErrorAction SilentlyContinue
Remove-Item c:\Users\bkuro\AppData\Local\Temp\ofw_*_reports -Recurse -ErrorAction SilentlyContinue

# Or find temp dir location
$env:TEMP  # Print temp directory location
```

### Reset Session (if needed)
The session is stored in memory, so restarting Flask clears it:
```
# Press Ctrl+C to stop Flask
# Then restart:
python app.py
```

## Success Criteria

✅ **Multi-year file processing is working** when:
1. All years show `[YEARLY TASK START]` messages
2. All years show `<<< [YEARLY TASK SUCCESS]` messages
3. Final ZIP download is generated with reports for all years
4. No timeout or exception messages in logs
5. Processing completes within 1-2 hours for typical 3-year file (36 months)

---

**Note**: The first time processing a file takes longer due to initial PDF parsing. Subsequent runs benefit from caching if the same file is re-processed.
