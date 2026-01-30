# Platform Copy Integration Summary

## Overview
Integrated platform ad copy generation into the video creation preview flow. When users generate a script preview, they now also get platform-specific ad copy (Meta, TikTok, YouTube) that can be edited alongside the script before final video generation.

## Architecture

### Flow
1. User clicks "Preview Script" in frontend
2. Frontend calls Job Gateway → Orchestration workflow
3. Orchestration calls Generate Script sub-workflow
4. Orchestration calls Generate Platform Copy sub-workflow
5. Both script and copy are written to Google Sheet (job status)
6. Frontend polls Job Gateway for status
7. When status = "script_ready", frontend displays both script AND copy editors

### Key Workflows (Dev Environment)

| Workflow | ID | Purpose |
|----------|-----|---------|
| [00] Job Gateway | `8sMB0en0dQvgIQA3` | Entry point, job status tracking |
| [01] Orchestrate Video Creation | `5pYAGAYJmg7qe3D7` | Main pipeline coordinator |
| [02] Generate Script | `qXvDgLuvrY7u6d7w` | Claude-powered script generation |
| [GENERATE] Platform Copy | `3ds9Y97qpRjTI63Y` | Claude-powered copy generation |

## Changes Made

### n8n Workflow Changes

#### [01] Orchestrate Video Creation
- Added flow: `If: Script Preview Mode` → `Prepare Copy Request` → `Call Copy Generator` → `Merge Copy into Config` → `Update Script Ready`
- `Prepare Copy Request` node: Builds copy request payload and stores `original_config`
- `Call Copy Generator` node: HTTP POST to copy generator webhook
- `Merge Copy into Config` node: Merges copy response back into config
- `Update Script Ready` node: Writes to Google Sheet with columns: `job_id`, `status`, `script_data`, `generated_copy`
- `Respond - Script Preview` node: Changed to `respondWith: 'firstIncomingItem'`
- `Call '[02] Generate Script'` node: Simplified to just workflow ID (removed broken workflowInputs)

#### [GENERATE] Platform Copy
- `Respond to Webhook` node: Changed to `respondWith: 'firstIncomingItem'` (was causing "Invalid JSON" errors)
- Uses `industry_copy_prompts` table for industry-specific copy prompts

#### [00] Job Gateway
- `Format Status Response` node: Added `generated_copy` field parsing from sheet
- `Respond to Webhook - Job Status` node: Changed to `respondWith: 'firstIncomingItem'`

### Database Changes

#### Dev Database (`shuttle.proxy.rlwy.net:56612`)
- Fixed `industry_copy_prompts` table: Updated `industry_id` from non-existent UUID to Automotive industry (`92771a9b-94f5-4bb1-86c1-1baa1e1677b6`)
- Table has prompts for: meta/ad, meta/organic, tiktok/ad, tiktok/organic, youtube/ad, youtube/organic

### Frontend Changes (`index.html`)

#### Added to `cachedWorkflowConfig` building (two locations ~line 1268 and ~1587):
```javascript
generated_copy: statusResult.generated_copy,  // Platform ad copy
```

#### Existing UI Elements (already present):
- `#copyEditorSection` - main copy editor container
- `#metaCopySection`, `#tiktokCopySection`, `#youtubeCopySection` - platform sections
- `populatePlatformCopy(generatedCopy)` function - populates copy fields
- `collectEditedCopy()` function - collects edited copy for submission

## Issues Encountered & Fixed

| Issue | Root Cause | Fix |
|-------|------------|-----|
| "Invalid JSON in response body" | Respond nodes using `={{ $json }}` or `={{ JSON.stringify($json) }}` | Changed to `respondWith: 'firstIncomingItem'` |
| No industry copy prompts found | `industry_copy_prompts` linked to wrong industry UUID | Updated industry_id to match Automotive |
| Script not populating in UI | `generated_copy` not included in frontend config | Added to `cachedWorkflowConfig` |
| "No workflow to execute" error | Broken `workflowInputs` in Execute Workflow node | Simplified node config to just workflow ID |

## Testing

### To Test Preview Flow:
1. Go to https://dev-demo.ad-pilot.ai
2. Select client, template, avatar, etc.
3. Click "Preview Script"
4. Should see script editor with Hook/Body/CTA
5. Should see copy editor sections for each platform below script

### To Debug:
- Check browser console for `Job status:` logs - should show `script_data` and `generated_copy`
- Check n8n execution logs for [01] Orchestrate Video Creation
- Verify Google Sheet has `generated_copy` column populated

## Current State
- Script preview: Working
- Copy generation: Working (workflow executes successfully)
- Copy display in UI: Should work after latest frontend push (adds `generated_copy` to config)

## Remaining Work / Known Issues
1. Verify copy actually displays in UI after latest frontend changes
2. Test the "Approve & Generate Video" flow with edited copy
3. The copy editor sections may need styling adjustments
4. Character count validators for each platform field

## API Credentials (Dev)

```
n8n API: https://ad-pilot-n8n-development.up.railway.app/api/v1
API Key: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIwNjcxZjY1Ni01MjY3LTRkNmMtYThiZC1lODYwMDFjZWM1ZjciLCJpc3MiOiJuOG4iLCJhdWQiOiJwdWJsaWMtYXBpIiwiaWF0IjoxNzY5NDcwMzkwfQ.dieiTjIjdwROkjLg3yhZvPdvoC-P5TdZsHg7BvgZQcU

Postgres: postgresql://postgres:SlhKtARznaTeNCcIYrYBmSPWNtlupkQr@shuttle.proxy.rlwy.net:56612/railway
```

## Useful Scripts

Scripts created during debugging are in `/tmp/`:
- `fix-copy-response.js` - Fix copy generator respond node
- `check-copy-generator.js` - Inspect copy generator workflow
- `fix-all-uuid-casts.js` - Fix UUID type casting in SQL queries
- `check-schema.js` - Check database schema
