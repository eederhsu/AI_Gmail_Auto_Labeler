// --- Configuration ---

// --- Pass 1: Categorization Definitions ---
const LABEL_DEFINITIONS = {
  financial:        'Emails related to bank statements, transactions, alerts, and communications from banks or other financial institutions.',
  immigration:      'Communications about U.S. visas and immigration processes.',
  newsletter:       'Regular newsletters and subscription digests.',
  updates:          'Subscription emails, mailing lists, blog updates, and other regular bulletins you’ve signed up for.',
  promotion:        'Unsolicited marketing messages, sales offers, and promotional spam.',
  shopping:         'Order confirmations, shipping notifications, delivery updates, returns, and invoices from online retailers.',
  housing:          'Apartment or building notices: rent, maintenance, utilities, cable, and related announcements.',
  investment:       'Stock-trading alerts, market summaries, trade confirmations from brokerage platforms.',
  news:             'Specialized news summaries and daily digest emails (e.g., “5 Things” bulletins).',
  accountSecurity:  'Account security alerts, registration confirmations, password resets, and 2FA codes.',
  travel:           'Travel-related bookings and confirmations (flights, hotels, car rentals, etc.).',
  career:           'Job-related communications: offers, contracts, onboarding, and visa-sponsorship updates.',
  other:            'Any email that doesn’t fit into the above categories (catch-all).',
};

const VALID_LABEL_NAMES = Object.keys(LABEL_DEFINITIONS);
const PROCESSED_LABEL_NAME = 'z-p1'; // Marker label for Pass 1 completion


// --- Pass 2: Importance / Action Definitions ---
const IMPORTANCE_LABEL_DEFINITIONS = {
  critical:  'Essential emails requiring immediate attention (e.g., visa docs, tax notices, fraud alerts).',
  urgent:    'Time-sensitive notifications needing prompt action (e.g., low-balance alerts, critical job messages).',
  archive:   'Items to retain for records but not urgent (e.g., past confirmations, delivered order receipts).',
  delete:    'Safe to delete after review (e.g., promotional spam, routine newsletters already read).',
};
const VALID_IMPORTANCE_LABELS = Object.keys(IMPORTANCE_LABEL_DEFINITIONS);
const IMPORTANCE_PROCESSED_LABEL_NAME = 'z-p2'; // Marker label for Pass 2 completion


// --- General Settings ---
const LLM_PROVIDER = 'openai';
const BATCH_SIZE = 1000; // Max threads per combined run (Adjust if timeouts occur)
const SCRIPT_TIMEZONE = Session.getScriptTimeZone();
const ENABLE_AUTO_ARCHIVE = false; // Set true to automatically archive 'toArchive' emails
const ENABLE_AUTO_DELETE = false; // *** DANGER *** Set true to automatically trash 'not_important_toDelete' emails

// --- Helper Functions ---

/**
 * Retrieves the OpenAI API key securely stored in Script Properties.
 */
function getApiKey() {
  const scriptProperties = PropertiesService.getScriptProperties();
  const apiKey = scriptProperties.getProperty('OPENAI_API_KEY');
  if (!apiKey) { throw new Error("FATAL: OPENAI_API_KEY not set in Script Properties."); }
  return apiKey;
}

/**
 * Gets a Gmail label object by name, creating it if it doesn't exist.
 */
function getOrCreateLabel(labelName) {
  let label = GmailApp.getUserLabelByName(labelName);
  if (!label) {
    try { Logger.log(`Creating label: ${labelName}`); label = GmailApp.createLabel(labelName); }
    catch (e) { Logger.log(`Error creating label "${labelName}": ${e}.`); throw e; }
  }
  return label;
}

/**
 * Extracts relevant text content, optionally including a context label.
 */
function getEmailTextContent(message, pass1LabelContext = null) {
  const subject = message.getSubject();
  let body = message.getPlainBody();
  const maxBodyLength = 1500;
  if (body.length > maxBodyLength) { body = body.substring(0, maxBodyLength) + "... (truncated)"; }
  const contextText = pass1LabelContext ? `\n(Category Guess: ${pass1LabelContext})` : "";
  return `Subject: ${subject}\nFrom: ${message.getFrom()}\nDate: ${message.getDate().toLocaleDateString()}${contextText}\n\n${body}`;
}

/**
 * Calls the configured LLM API (OpenAI).
 */
function callLLM(prompt) {
  const apiKey = getApiKey();
  let apiUrl, payload, options;
  if (LLM_PROVIDER === 'openai') {
    apiUrl = PropertiesService.getScriptProperties().getProperty('LLM_API_ENDPOINT') || 'https://api.openai.com/v1/chat/completions';
    payload = JSON.stringify({ model: "gpt-4.1-mini", messages: [{ role: "user", content: prompt }], temperature: 0.1, max_tokens: 50 });
    options = { method: 'post', contentType: 'application/json', headers: { 'Authorization': 'Bearer ' + apiKey }, payload: payload, muteHttpExceptions: true };
  } else { Logger.log(`LLM provider "${LLM_PROVIDER}" not configured.`); return null; }

  try {
    const response = UrlFetchApp.fetch(apiUrl, options); const responseCode = response.getResponseCode(); const responseBody = response.getContentText();
    if (responseCode === 200) {
      const jsonResponse = JSON.parse(responseBody);
      if (jsonResponse.choices && jsonResponse.choices.length > 0 && jsonResponse.choices[0].message && jsonResponse.choices[0].message.content) {
        let cleanedLabel = jsonResponse.choices[0].message.content.trim().replace(/["'.]/g, '');
        const allValidLabels = [...VALID_LABEL_NAMES, ...VALID_IMPORTANCE_LABELS];
        for (const validLabel of allValidLabels) { if (cleanedLabel.includes(validLabel)) { return validLabel; } }
        return cleanedLabel;
      } else { Logger.log(`LLM API response format unexpected: ${responseBody}`); return null; }
    } else {
       if (responseCode === 429) Logger.log("LLM Error: 429 - Rate limit hit."); else if (responseCode === 401) Logger.log("LLM Error: 401 - Auth failed (API Key?)."); else if (responseCode === 400) Logger.log(`LLM Error: 400 - Bad Request: ${responseBody}`); else Logger.log(`LLM Error: ${responseCode} - ${responseBody}`); return null;
    }
  } catch (error) { Logger.log(`Exception calling UrlFetchApp.fetch: ${error}`); return null; }
}

/**
 * Parses various date specifiers ("today", "yesterday", Date object, "YYYY/MM/DD" string)
 * into a Date object representing the start of that day in the script's timezone.
 * Handles undefined/null input gracefully.
 * @param {string | Date | undefined | null} dateSpecifier The date identifier.
 * @return {Date | null} A valid Date object or null if parsing fails or input is invalid.
 */
function parseDateSpecifier(dateSpecifier) {
    if (dateSpecifier === undefined || dateSpecifier === null) {
        Logger.log("Error inside parseDateSpecifier: Received undefined or null input. Check how the function calling this is getting its arguments.");
        return null;
    }
    let targetDate = new Date();
    try {
        if (dateSpecifier instanceof Date) { targetDate = dateSpecifier; }
        else if (typeof dateSpecifier === 'string') {
            const specifierLower = dateSpecifier.toLowerCase();
            if (specifierLower === 'today') {}
            else if (specifierLower === 'yesterday') { targetDate.setDate(targetDate.getDate() - 1); }
            else if (/^\d{4}\/\d{2}\/\d{2}$/.test(dateSpecifier)) {
                const parts = dateSpecifier.split('/'); targetDate = new Date(parseInt(parts[0], 10), parseInt(parts[1], 10) - 1, parseInt(parts[2], 10));
                if (isNaN(targetDate.getTime())) throw new Error("Invalid date from YYYY/MM/DD");
            } else { throw new Error(`Invalid date string format: "${dateSpecifier}".`); }
        } else { throw new Error(`Invalid date specifier type: ${typeof dateSpecifier}`); }
        return new Date(targetDate.getFullYear(), targetDate.getMonth(), targetDate.getDate());
    } catch (e) { Logger.log(`Error parsing date "${dateSpecifier}": ${e}`); return null; }
}

/**
 * Calculates 'after:' and 'before:' date strings needed for a Gmail query.
 */
function getDateQueryStrings(startDateSpecifier, endDateSpecifier) {
  Logger.log(`Entering getDateQueryStrings with startDateSpecifier: ${startDateSpecifier}, endDateSpecifier: ${endDateSpecifier}`);
  const startDate = parseDateSpecifier(startDateSpecifier);
  const endDate = parseDateSpecifier(endDateSpecifier);
  if (!startDate || !endDate) { Logger.log("getDateQueryStrings failed: Invalid start or end date returned from parseDateSpecifier."); return null; }
  if (endDate < startDate) { Logger.log(`getDateQueryStrings failed: End date (${endDate.toLocaleDateString()}) is before start date (${startDate.toLocaleDateString()}).`); return null; }
  const beforeDate = new Date(endDate); beforeDate.setDate(endDate.getDate() + 1);
  const startDateStr = Utilities.formatDate(startDate, SCRIPT_TIMEZONE, "yyyy/MM/dd");
  const beforeDateStr = Utilities.formatDate(beforeDate, SCRIPT_TIMEZONE, "yyyy/MM/dd");
  const endDateLogStr = Utilities.formatDate(endDate, SCRIPT_TIMEZONE, "yyyy/MM/dd");
  Logger.log(`Querying date range: after:${startDateStr} (inclusive) up to before:${beforeDateStr} (includes emails up to end of ${endDateLogStr})`);
  return { startDateStr: startDateStr, endDateStr: beforeDateStr };
}


// --- COMBINED Main Processing Function (Pass 1 & Pass 2) ---

function processInboxCombined(startDateSpecifier, endDateSpecifier) {
  // *** NEW: Explicit check for undefined arguments ***
  if (startDateSpecifier === undefined || endDateSpecifier === undefined) {
      const errorMsg = "FATAL ERROR in processInboxCombined: Received undefined start or end date specifier. Ensure triggers/manual runs use wrapper functions (e.g., 'runProcessTodaysCombined'). Aborting.";
      Logger.log(errorMsg);
      // Optional: throw new Error(errorMsg); // Make error more prominent
      return; // Stop execution cleanly
  }
  // *** END NEW CHECK ***

  Logger.log(`Entering processInboxCombined with startDateSpecifier: ${startDateSpecifier}, endDateSpecifier: ${endDateSpecifier}`);
  Logger.log(`Typeof startDateSpecifier: ${typeof startDateSpecifier}, typeof endDateSpecifier: ${typeof endDateSpecifier}`);
  const startTime = new Date();
  Logger.log(`--- Starting COMBINED Processing for range: ${startDateSpecifier} to ${endDateSpecifier} ---`);

  // --- Prepare All Labels ---
  const pass1ProcessedLabel = getOrCreateLabel(PROCESSED_LABEL_NAME); // z-p1
  const pass2ProcessedLabel = getOrCreateLabel(IMPORTANCE_PROCESSED_LABEL_NAME); // z-p2
  let categoryLabelsMap, importanceLabelsMap;
  try {
      categoryLabelsMap = VALID_LABEL_NAMES.reduce((map, name) => { map[name] = getOrCreateLabel(name); return map; }, {});
      importanceLabelsMap = VALID_IMPORTANCE_LABELS.reduce((map, name) => { map[name] = getOrCreateLabel(name); return map; }, {});
      Logger.log(`All labels prepared.`);
  } catch (e) { Logger.log(`COMBINED Critical error preparing labels. Aborting. Error: ${e}`); return; }

  // --- Get Date Range ---
  const dateStrings = getDateQueryStrings(startDateSpecifier, endDateSpecifier);
  if (!dateStrings) { Logger.log("COMBINED Invalid date range received. Aborting function."); return; }
  const { startDateStr, endDateStr } = dateStrings;

  // --- Define Query ---
  const query = `in:inbox after:${startDateStr} before:${endDateStr} -label:${IMPORTANCE_PROCESSED_LABEL_NAME}`;
  Logger.log(`COMBINED Query (finding emails needing Pass 2): ${query}`);

  let start = 0; let threadsProcessedInRun = 0; let threadsFetchedCount = 0;
  const maxThreadsToProcess = BATCH_SIZE; const fetchBatchSize = 300;

  // --- Build Prompt Definitions ---
  let pass1DefinitionsPrompt = "Available Category Labels and Definitions:\n";
  for (const labelName in LABEL_DEFINITIONS) { pass1DefinitionsPrompt += `- ${labelName}: ${LABEL_DEFINITIONS[labelName]}\n`; }
  pass1DefinitionsPrompt += `\nWhich category label from the list [${VALID_LABEL_NAMES.join(', ')}] best fits this email?`;
  let pass2DefinitionsPrompt = "Available Importance/Action Labels and Definitions:\n";
  for (const labelName in IMPORTANCE_LABEL_DEFINITIONS) { pass2DefinitionsPrompt += `- ${labelName}: ${IMPORTANCE_LABEL_DEFINITIONS[labelName]}\n`; }
  pass2DefinitionsPrompt += `\nWhich label from the list [${VALID_IMPORTANCE_LABELS.join(', ')}] best describes the importance/action for this email?`;

  // --- Processing Loop ---
  while (threadsProcessedInRun < maxThreadsToProcess) {
    let executionTimeLeft = (6 * 60 * 1000) - (new Date() - startTime);
    if (executionTimeLeft < 45000) { Logger.log(`COMBINED Nearing time limit (${(executionTimeLeft/1000).toFixed(0)}s left). Processed ${threadsProcessedInRun}. Stopping.`); break; }

    // Fetch Batch
    const threadsToFetchNow = Math.min(fetchBatchSize, maxThreadsToProcess - threadsProcessedInRun);
    if (threadsToFetchNow <= 0) break;
    let threads; try { threads = GmailApp.search(query, start, threadsToFetchNow); threadsFetchedCount = threads.length; Logger.log(`COMBINED Fetched ${threadsFetchedCount} threads (index ${start}). Total processed this run: ${threadsProcessedInRun}`); } catch (e) { Logger.log(`COMBINED Gmail search error: ${e}. Stopping.`); break; }
    if (threadsFetchedCount === 0) { Logger.log("COMBINED No more matching threads found for this range/batch."); break; }

    let currentBatchTimeLeft = executionTimeLeft; // Track time within batch

    // Process Batch
    for (const thread of threads) {
      if (threadsProcessedInRun >= maxThreadsToProcess) break;
      let message; let pass1LabelAppliedThisRun = null;
      try {
          message = thread.getMessages()[thread.getMessageCount() - 1];
          const currentLabels = thread.getLabels().map(l => l.getName());
          if (currentLabels.includes(IMPORTANCE_PROCESSED_LABEL_NAME)) { Logger.log(`Skipping thread "${message.getSubject()}" - already has ${IMPORTANCE_PROCESSED_LABEL_NAME}`); continue; }

          // === Pass 1 Logic (Conditional) ===
          if (!currentLabels.includes(PROCESSED_LABEL_NAME)) {
            Logger.log(`-- Running Pass 1 (Categorization) for: "${message.getSubject()}"`);
            const emailContentP1 = getEmailTextContent(message);
            const promptP1 = `Analyze the email and assign exactly ONE category label based on the definitions. Respond ONLY with the single chosen label name.\n\n${pass1DefinitionsPrompt}\n\n--- Email Start ---\n${emailContentP1}\n--- Email End ---`;
            const llmResponseP1 = callLLM(promptP1);
            if (llmResponseP1 && VALID_LABEL_NAMES.includes(llmResponseP1)) {
              const targetLabelObject = categoryLabelsMap[llmResponseP1]; Logger.log(`   P1 Applying Category: "${llmResponseP1}"`); thread.addLabel(targetLabelObject); thread.addLabel(pass1ProcessedLabel); pass1LabelAppliedThisRun = llmResponseP1;
            } else {
              if (!llmResponseP1) Logger.log(`   P1 LLM FAIL. Applying only '${PROCESSED_LABEL_NAME}'.`); else Logger.log(`   P1 LLM UNRECOGNIZED "${llmResponseP1}". Applying only '${PROCESSED_LABEL_NAME}'.`); thread.addLabel(pass1ProcessedLabel);
            }
            Utilities.sleep(200);
          } else { Logger.log(`-- Skipping Pass 1 for: "${message.getSubject()}" (already has ${PROCESSED_LABEL_NAME})`); }

          // === Pass 2 Logic (Always runs if z-p2 missing) ===
          Logger.log(`-- Running Pass 2 (Importance) for: "${message.getSubject()}"`);
          let contextLabelForP2 = pass1LabelAppliedThisRun;
          if (!contextLabelForP2) { const existingP1Label = currentLabels.find(l => VALID_LABEL_NAMES.includes(l)); if (existingP1Label) contextLabelForP2 = existingP1Label; }
          const emailContentP2 = getEmailTextContent(message, contextLabelForP2);
          const promptP2 = `Analyze the importance/action needed for the email based on the definitions. Respond with the single chosen label name.\n\n${pass2DefinitionsPrompt}\n\n--- Email Start ---\n${emailContentP2}\n--- Email End ---`;
          const llmResponseP2 = callLLM(promptP2);
          if (llmResponseP2 && VALID_IMPORTANCE_LABELS.includes(llmResponseP2)) {
            const targetLabelObject = importanceLabelsMap[llmResponseP2]; Logger.log(`   P2 Applying Importance: "${llmResponseP2}"`); thread.addLabel(targetLabelObject); thread.addLabel(pass2ProcessedLabel);
            if (llmResponseP2 === 'toArchive' && ENABLE_AUTO_ARCHIVE) { Logger.log(`   P2 Archiving.`); thread.moveToArchive(); }
            else if (llmResponseP2 === 'not_important_toDelete' && ENABLE_AUTO_DELETE) { Logger.log(`   P2 MOVING TO TRASH.`); thread.moveToTrash(); }
            else if (llmResponseP2 === 'important_toRead') { Logger.log(`   P2 Marking Important.`); thread.markImportant(); }
          } else {
            if (!llmResponseP2) Logger.log(`   P2 LLM FAIL. Applying only '${IMPORTANCE_PROCESSED_LABEL_NAME}'.`); else Logger.log(`   P2 LLM UNRECOGNIZED "${llmResponseP2}". Applying only '${IMPORTANCE_PROCESSED_LABEL_NAME}'.`); thread.addLabel(pass2ProcessedLabel);
          }
          threadsProcessedInRun++; Utilities.sleep(400);
      } catch (e) {
          Logger.log(`COMBINED Error processing thread (Subject: ${message ? message.getSubject() : 'N/A'}): ${e}. Skipping thread.`);
          try { if (thread && !thread.getLabels().map(l=>l.getName()).includes(PROCESSED_LABEL_NAME)) thread.addLabel(pass1ProcessedLabel); } catch (labelErr) { Logger.log(`Failed applying z-p1 after error: ${labelErr}`)}
          try { if (thread && !thread.getLabels().map(l=>l.getName()).includes(IMPORTANCE_PROCESSED_LABEL_NAME)) thread.addLabel(pass2ProcessedLabel); } catch (labelErr) { Logger.log(`Failed applying z-p2 after error: ${labelErr}`)}
          threadsProcessedInRun++; Utilities.sleep(150);
      }
       currentBatchTimeLeft = (6 * 60 * 1000) - (new Date() - startTime); if (currentBatchTimeLeft < 30000) { Logger.log(`COMBINED Inner loop nearing time limit (${(currentBatchTimeLeft/1000).toFixed(0)}s left). Processed ${threadsProcessedInRun}. Stopping.`); break; }
    } // End for
    start += threadsFetchedCount; if (threadsFetchedCount < threadsToFetchNow) break; if (currentBatchTimeLeft < 30000) break;
  } // End while
  const endTime = new Date(); const duration = (endTime - startTime) / 1000;
  Logger.log(`--- Finished COMBINED Processing for range ${startDateSpecifier} to ${endDateSpecifier}. Processed ${threadsProcessedInRun} threads in ${duration.toFixed(1)}s. ---`);
}


// --- Wrapper Functions for Manual Execution & Triggers ---

/** Runs Combined Pass 1 & 2 for TODAY */
function runProcessTodaysCombined() {
    processInboxCombined('today', 'today');
}
/** Runs Combined Pass 1 & 2 for YESTERDAY */
function runProcessYesterdaysCombined() {
    processInboxCombined('yesterday', 'yesterday');
}
/** Runs Combined Pass 1 & 2 for a specific DATE RANGE. Modify dates below before running manually. */
function runProcessDateRangeCombined() {
  // *** DEFINE YOUR DATE RANGE HERE (Format: YYYY/MM/DD) ***
  const startDate = "2021/01/01"; // Example Start Date
  const endDate = "2024/12/31";   // Example End Date
  // *******************************************************
  Logger.log(`Manually triggering COMBINED processing for range: ${startDate} to ${endDate}`);
  processInboxCombined(startDate, endDate);
}
