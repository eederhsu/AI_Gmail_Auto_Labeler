# AI Gmail Auto-Labeler Script

A Google Apps Script to automatically apply category and importance labels to incoming Gmail messages based on customizable definitions. Perfect for organizing your inbox with minimal manual effort.

---

## Features

* Define your own **email categories** (e.g., financial, newsletter, travel, etc.)
* Define **importance actions** (e.g., critical, urgent, archive, delete)
* Automatically applies labels to matching messages
* Fully customizable label definitions to suit your workflow

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [Label Definitions](#label-definitions)
5. [Deploying the Script](#deploying-the-script)
6. [Usage](#usage)
7. [Contributing](#contributing)
8. [License](#license)

---

## Prerequisites

* A Google account with Gmail enabled
* Access to Google Apps Script
* Basic familiarity with JavaScript/Google Apps Script editor

---

## Usage

Follow these steps to set up the script in Google Apps Script:

1. Go to [script.google.com](https://script.google.com/) and click **New project**.
2. Name your project (e.g. `Gmail Auto-Labeler`).
3. In the left sidebar, click **+** > **Script file**, name it `Code.gs`.
4. Open the `Code.gs` file in this repo, copy its contents, and paste into the editor.
5. Add your API key:

   * Click the gear icon (**Project Settings**).
   * Under **Script properties**, click **Add script property**.
   * Set the property name to `OPENAI_API_KEY` and the value to your API key.
6. Save your project (**File > Save**).

(Optional) If you prefer to work locally:

```bash
git clone https://github.com/yourusername/gmail-auto-labeler.git
cd gmail-auto-labeler
```

---

## Configuration

1. Open the `Code.gs` file in the Apps Script editor.
2. Edit the label definitions at the top of the file:

   ```js
   const LabelDefinitions = {
     financial:       'Emails related to bank statements, transactions, and financial alerts.',
     immigration:     'Communications about visas and immigration.',
     newsletter:      'Subscription newsletters and digests.',
     updates:         'Mailing lists, blog updates, and bulletins.',
     promotion:       'Marketing messages and promotional spam.',
     shopping:        'Order confirmations, shipping notices, and invoices.',
     housing:         'Building notices and utility updates.',
     investment:      'Stock-trading alerts and market summaries.',
     news:            'Daily news summaries and bulletins.',
     accountSecurity: 'Account alerts and 2FA codes.',
     travel:          'Flight, hotel, and rental confirmations.',
     career:          'Job offers, contracts, and onboarding messages.',
     other:           'Catch-all for uncategorized emails.',
   };

   const ImportanceDefinitions = {
     critical: 'Requires immediate attention (e.g., fraud alerts, tax notices).',
     urgent:   'Time-sensitive actions needed (e.g., low balance alerts).',
     archive:  'Keep for records but not urgent (e.g., past receipts).',
     delete:   'Safe to remove after review (e.g., promotional spam).',
   };
   ```
3. Adjust the definitions or add new categories as needed.

---

## Label Definitions

* **Categories** map message content or sender patterns to Gmail labels.
* **Importance** levels decide whether to mark, archive, or delete messages.

Customize both objects to match your personal or team workflow.

---

## Deploying the Script

1. **Authorize** the script:

   * In the Apps Script editor, click **Run > runMain** (or your entry function) and grant permissions.
2. **Set up trigger**:

   * In the editor, click **Triggers (clock icon)**
   * Click **+ Add Trigger**
   * Choose the entry function (`runAutoLabeler`)
   * Event source: **Time-driven**
   * Select frequency (e.g., every 5 minutes or hourly)
   * Save

> **Tip:** You can also trigger on incoming mail using Gmail push notifications, but time-driven is simplest.

---

## Usage

* The script will scan your inbox at each trigger interval,
* It applies labels defined in your configuration based on message content, sender, and subject.
* Check Gmail labels sidebar to see messages categorized automatically.

---

## Contributing

Contributions are welcome! Please open an issue or submit a pull request:

1. Fork the repo
2. Create a feature branch (`git checkout -b feature/YourFeature`)
3. Commit your changes (`git commit -m 'Add new label category'`)
4. Push to the branch (`git push origin feature/YourFeature`)
5. Open a Pull Request

---

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.
