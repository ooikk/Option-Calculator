# Option-Calculator
**Online Option Calculator**

## Google Apps Script (GAS) Integration
**1. Deployment Method:** The HTML must be served via the **doGet()** function in Google Apps Script and accessed via the Web App URL. If you are opening the HTML file locally on your computer, google.script.run will fail because it can't find the Google server context.

**2. API Rate Limiting:** Yahoo Finance sometimes throttles UrlFetchApp requests from Google’s IP addresses.

**3. Permissions:** You may need to re-authorize the script after adding UrlFetchApp.

## The Integrated Solution

**1. The Server-Side (Code.gs)**

This handles the heavy lifting. I’ve added a try-catch block and a strict 5-second timeout to prevent the "hanging" state. Paste below code into Google Sheet apps

```
// This function serves the HTML page
function doGet() {
  return HtmlService.createHtmlOutputFromFile('Index')
    .setTitle('Option Visualizer')
    .setXFrameOptionsMode(HtmlService.XFrameOptionsMode.ALLOWALL);
}

// The Scraper Function
function fetchTickerPrice(symbol) {
  if (!symbol) return null;
  
  // Clean symbol to get the ticker (e.g., KWEB26... -> KWEB)
  const ticker = symbol.match(/^[A-Z]+/)[0];
  const url = `https://query1.finance.yahoo.com/v8/finance/chart/${ticker}`;
  
  try {
    const response = UrlFetchApp.fetch(url, {
      'muteHttpExceptions': true,
      'connectTimeoutInMilliseconds': 5000, // 5 second cap
      'readTimeoutInMilliseconds': 5000
    });
    
    if (response.getResponseCode() === 200) {
      const data = JSON.parse(response.getContentText());
      return data.chart.result[0].meta.regularMarketPrice;
    }
  } catch (e) {
    console.error("Scrape Error: " + e.toString());
  }
  return null; // Return null so the UI can stop the loading state
}
```

**2. The Client-Side (Index.html) - Focus on the Sync Bridge**

In your HTML, ensure the scrapePrice function handles the "failure" case so the button stops spinning even if the data doesn't come back.

```
function scrapePrice() {
    const symbols = document.querySelectorAll('.symbol-input');
    if (symbols.length === 0) return;
    
    const rawSymbol = symbols[0].value.toUpperCase();
    const btn = document.getElementById('syncBtn');
    
    // Start Loading State
    btn.classList.add('animate-spin', 'opacity-50');

    // Calling the .gs function
    google.script.run
        .withSuccessHandler((price) => {
            btn.classList.remove('animate-spin', 'opacity-50');
            if (price) {
                document.getElementById('currentPrice').value = price.toFixed(2);
                document.getElementById('targetPrice').value = price.toFixed(2); // Keep in sync as per image_c19660.png
                calculate();
            } else {
                alert("Could not fetch price. Try entering manually.");
            }
        })
        .withFailureHandler((err) => {
            btn.classList.remove('animate-spin', 'opacity-50');
            console.error("Script Bridge Error:", err);
            alert("Connection error to Google Script.");
        })
        .fetchTickerPrice(rawSymbol);
}
```
## Steps to integrate to Google Sheet
**1.0 Authorize:**

In the Apps Script editor, click the **"Run"** button for **fetchTickerPrice** once manually. It will prompt you for permissions to **"Connect to an external service."** You must approve this.

**2.0 Deploy as Web App:**
- Click **Deploy > New Deployment.**
- Select **Type: Web App.**
- Set **Execute as: Me.**
- Set **Who has access: Anyone (or yourself).**

**3.0 Use the Provided URL:**

Only use the URL provided by the deployment to view your visualizer. If you test via "Test deployments," ensure you refresh the page after any code change.


