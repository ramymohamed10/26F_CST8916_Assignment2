# Assignment 2 · ShopStream promotion analytics

**CST8916 · Remote Data and Real-time Applications · Fall 2026**

| Weight | Due date | Submit on Brightspace |
| --- | --- | --- |
| 10% of the final grade | See the Assignment 2 deadline in Brightspace | URL of your private GitHub repository |

ShopStream is running a promotion. The store manager opens the Week 10 dashboard and sees individual shoppers adding items to their carts. **Which device types are generating the most activity? When do shoppers start opening product details in quick succession?**

Extend the Week 10 Flask app so its events carry device information. Use Azure Stream Analytics to answer both questions from the live `clickstream` event hub. Display the results on the store’s deployed `/dashboard` page.

## 1. Start from the two labs

| Source | What to reuse |
| --- | --- |
| [Week 10 · ShopStream and Event Hubs](https://github.com/ramymohamed10/26F_CST8916_Week10-Event-Hubs-Lab) | Start your private assignment repository from a copy of this app. Follow a store action from `static/store.js` through `POST /track` in `app.py`. In `shop.py`, `normalize_event()` builds the JSON that Flask actually publishes. |
| [Week 11 · The campus concert ticket rush](https://github.com/ramymohamed10/26F_CST8916_Week11-Stream-Analytics-Lab) | Reuse the workflow for testing a query with a recording, starting a job, and inspecting its output. Apply it to ShopStream’s fields. The store calls its UTC event time `timestamp` and records opening product details as `event_type = 'product_click'`. |

Use your Azure for Students subscription and a dedicated assignment resource group. If you cleaned up the labs, recreate only the resources your design uses.

## 2. Enrich the events

Add three string fields to every new event the store publishes. Choose how to identify the device, browser, and operating system. Use a consistent value such as `unknown` when a detail is unavailable.

| Field | Example values |
| --- | --- |
| `deviceType` | `desktop`, `mobile`, `tablet` |
| `browser` | `Chrome`, `Firefox`, `Safari` |
| `os` | `Windows`, `macOS`, `Android`, `iOS` |

Check the result at the Event Hub input. Open a product on a desktop, then use a phone or a browser’s mobile-device emulation. Inspect a received event from each. Both should have the three new fields alongside their Week 10 fields, including `event_id`, `event_type`, and the server-assigned `timestamp`. The same enrichment should reach checkout and purchase events.

## 3. Write and test the Stream Analytics queries

Create one Azure Stream Analytics job. Configure `clickstream` as its JSON Event Hub input and add the outputs you will use in section 4. Use `TIMESTAMP BY [timestamp]` on the ShopStream input so both calculations follow the time Flask recorded. Save the complete job query, including its output aliases, in `analytics.sql` in your repository.

| Manager’s question | Required result |
| --- | --- |
| Which devices are generating traffic? | For each fixed, non-overlapping 10-second window, count **all store events** by `deviceType`. Return the UTC window end, device type, and event count. A device with no events in a window needs no row. |
| When does interest in products spike? | Raise an alert when **at least three** `product_click` events occur across the store within **any 10 seconds**, including bursts that cross a fixed reporting boundary. Return the UTC alert time and the qualifying click count. |

Choose the window and grouping for each result. Think about whether clicks from two shoppers on different devices should contribute to the same alert. [Week 11 SAQL reference](https://github.com/ramymohamed10/26F_CST8916_Week11-Stream-Analytics-Lab/blob/main/SAQL-Reference.md)

### Check your answers against a recording

With the job stopped, select its Event Hub input in the query editor, upload the [seven-event ShopStream recording](sample/shopstream-promotion.json) as sample input, and use **Test query**. Compare both outputs with these checks.

| Window end (UTC) | Desktop events | Mobile events |
| --- | ---: | ---: |
| 09:00:10 | 2 | 2 |
| 09:00:20 | 1 | 1 |
| 09:00:30 | 0 | 1 |

The first alert should appear at **09:00:12 UTC** with **3 product clicks**. Find events `e03`, `e04`, and `e05` in the recording and explain why they trigger one alert even though they appear in two reporting windows. The cart action at 09:00:16 counts toward desktop traffic. The isolated product click at 09:00:24 should produce no alert.

## 4. Connect the results to the deployed dashboard

Add a device traffic view and a recent spike alert view to `/dashboard`. Show the UTC time and count for each result so the manager can tell which interval it describes. Both views must update automatically from **live Stream Analytics output**. Keep the Week 10 store, checkout, and recent-event feed working.

- **Choose an output.** One route from Week 11 is a private Blob container. Configure two job outputs pointing to separate paths for traffic and alerts. Have Flask read the line-separated JSON and serve the rows to the dashboard. Keep credentials in server-side app settings or use managed identity. You may use another Stream Analytics output if you document how Flask receives both result types. [Blob output settings](https://learn.microsoft.com/en-us/azure/stream-analytics/blob-storage-azure-data-lake-gen2-output)
- **Plan for both readers.** The Week 10 Flask consumer already reads `clickstream` using `$Default`. On Basic, each hub has one consumer group. One workable Basic design gives Stream Analytics the `clickstream` input and adds a pass-through output that forwards the original events to a second hub. Point only the Flask **consumer** at that second hub. The Flask **producer** continues to publish to `clickstream`. If you already use Standard, you can keep the raw feed as built and give Stream Analytics its own consumer group on `clickstream`. [Event Hubs tier limits](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-quotas)
- **Verify a live run.** Deploy to your own Azure App Service as in Week 10. Start the analytics job with output start time **Now** and wait for **Running**. Open the deployed store and generate fresh activity from two device types. Open product details three times in quick succession to trigger an alert. Allow time for windows to close and for output delivery, particularly with Blob batching. Match a dashboard result to its output row using the UTC time and count.

## 5. Document and demonstrate your work

Your repository’s `README.md` must include the following items.

- An architecture diagram tracing a store action to both the recent-event feed and the two analytics views. Label the hubs, consumer groups, job outputs, and dashboard data source.
- Your reasoning for the enrichment method, window choices, and route from Stream Analytics to Flask. Use the three clicks in the recording to explain the alert.
- Reproduction steps covering resources, input and output aliases, required app setting names, deployment, and how to trigger a live spike. Use placeholders for credentials.
- Your **unlisted YouTube video link**.

Record **up to five minutes**. Audio is optional. You can capture the sample test while the job is stopped, then record the live sections after the job starts. Show the following evidence with credentials hidden.

1. Open the store on its Azure App Service URL. Generate activity from two device types and trigger a burst of product clicks.
2. Show the enriched events received from Event Hubs. Identify the device, browser, and OS on each.
3. Show your SAQL and the sample test results for the device counts and the cross-boundary alert.
4. Show both views updating on the deployed dashboard. Match one live alert to the same time and click count in the job’s output.

Submit the **private GitHub repository URL** on Brightspace. Include the app changes, `analytics.sql`, and your README. Add **`ramymohamed10`** as a collaborator before the deadline and verify the video link opens. Submissions without collaborator access at marking time will not be accepted. After recording and checking your evidence, stop the job and delete your dedicated assignment resource group to avoid further charges.

## 6. Marking rubric

| Component | What earns the marks | Weight |
| --- | --- | ---: |
| Event enrichment | All three fields reach Event Hubs on store events, with meaningful values for the demonstrated devices. | 15% |
| Stream Analytics and dashboard | Event-time device summaries (10%), cross-boundary spike detection (10%), and both live outputs displayed on the deployed dashboard (15%). | 35% |
| Documentation | Architecture diagram (5%), design decisions and sample interpretation (5%), and reproducible setup (5%). | 15% |
| Video demonstration | Deployed activity and enriched input (10%), both sample query results (10%), and live dashboard updates with an alert matched to its output (15%). | 35% |
| **Total** | | **100%** |

## Academic integrity

The course permits and encourages generative AI assistance. If you use it, disclose its use in a comment at the top of each affected code file. You are responsible for testing and explaining all code and queries you submit. Direct assignment questions to [Ramy Mohamed](mailto:mohamer@algonquincollege.com).
