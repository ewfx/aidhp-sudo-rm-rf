## Architecture : SUDO-RM-RF

# sudo-rm-rf Architecture

Below is a detailed design and architectural documentation for your recommendation system. This document outlines the data flows, components, and scheduler processes. It also includes Mermaid diagrams to visualize the overall system and database schema.

---

## 1. System Overview

The system consists of two main scheduled processes:

- **End-of-Day (EOD) Transaction Validation Scheduler:**
    
    Runs every end-of-day (EOD) to loop through each transaction of the day and mark it as *valid* or *invalid* for recommendation. This flag is stored in the `transactions` collection.
    
- **Bi-Weekly Recommendation Scheduler:**
    
    Runs every two weeks to process a customer’s transactions from that period. For each customer:
    
    1. Retrieve the customer’s profile (including the list of products the customer is already using – **P1**).
    2. From the customer’s segment (via `segment_id`), query the `products` collection to get the list of all recommendable products – **P2**.
    3. Compute the difference **P2 - P1** to get products that the customer is not currently using.
    4. Collect all valid transactions within the two-week timeframe.
    5. Construct a prompt that combines the transaction history and the available products (from **P2 - P1**).
    6. Send this prompt to the LLM (via OpenAI Chat Completions) to obtain a recommendation.
    7. Parse the JSON response and update the recommendation result (including the chosen product) along with the corresponding transaction ID used as context.

---

## 2. Database Schema Context

### Customer Schema

![Customer][customer-screenshot]

### Segment Schema

![Segment][segment-screenshot]

### Product Schema

![Product][product-screenshot]

### Transaction Schema

![Transaction][transaction-screenshot]

---

## 3. Architectural Flow

### A. End-of-Day (EOD) Transaction Validation Scheduler

1. **Trigger:** Runs at EOD.
2. **Process:**
    - Query all transactions for the day.
    - Apply validation rules (e.g., check for anomalies, insufficient funds, etc.).
    - Update each transaction with a flag: `is_processed_for_recommendation` = `true` (valid) or leave as `false` (or mark invalid with an additional flag, if desired).

**Mermaid Flow Diagram for EOD Scheduler:**

![flow][flow-screenshot]

### B. Bi-Weekly Recommendation Scheduler

1. **Trigger:** Runs every two weeks.
2. **Process for each Customer:**
    - **Step 1:** Retrieve all transactions (with valid flag) in the past two weeks.
    - **Step 2:** Fetch customer profile (including products already used, **P1**).
    - **Step 3:** Query the `products` collection by `segment_id` to retrieve **P2** (all available products for that customer segment).
    - **Step 4:** Compute the difference: **P2 - P1** (available products not used).
    - **Step 5:** Construct a context/prompt that includes:
        - Summary of the customer's transactions in the period.
        - The list of available products (from **P2 - P1**).
    - **Step 6:** Call the LLM API using OpenAI Chat Completions with:
        - Model: `deepseek-r1`
        - Temperature: set (e.g., 0.7)
        - A system message with instructions (output as JSON)
        - A user message containing the context.
    - **Step 7:** Parse the JSON response to extract:
        - `transaction_id` used for recommendation context.
        - Recommended product.
        - Reason for recommendation.
    - **Step 8:** Update the recommendation results in the system (log, update customer profile, or send notification).

**Mermaid Sequence Diagram for Bi-Weekly Scheduler:**

![Biweekly][biweekly-screenshot]

---

## 4. Flask API Setup for Recommendation Service

In your Flask application (as described in previous parts), you would expose endpoints (for example, to manually trigger recommendations or view analytics). The controller will call the service layer, which in turn calls the LLM API using your configured `openai_util`.

### Example API Flow (Controller → Service → LLM)

![APIFLOW][apiflow-screenshot]
---

## 5. Additional Considerations

- **Error Handling & Logging:**
    
    Ensure each scheduler and API endpoint logs errors and key actions (e.g., if the LLM returns non-JSON output or if no transactions are found).
    
- **Scalability:**
    
    Consider processing transactions in batches if the volume is high, and use asynchronous processing (e.g., Celery) for schedulers.
    
- **Security & Configurations:**
    
    Use environment variables for API keys, DB credentials, and other sensitive settings (e.g., in `.env`). Use Flask security best practices for API endpoints.
    
- **Testing:**
    
    Create unit tests for each service function (data retrieval, prompt construction, LLM parsing) and integration tests for the full API flow.
    
- **Extensibility:**
    
    The design separates controllers, services, and the recommendation engine, so you can easily extend each component independently (for example, supporting multiple LLM models or additional recommendation rules).
    

---

## Conclusion

This design leverages scheduled batch processing to validate transactions and generate product recommendations over a two-week window. It uses a layered approach:

- **Data Layer:** MongoDB collections for Customers, Segments, Products, and Transactions.
- **Service Layer:** Business logic for filtering transactions, computing product differences, and constructing prompts.
- **Recommendation Engine:** A dedicated module that calls OpenAI Chat Completions.
- **API Layer:** Flask controllers exposing endpoints for querying transactions, analyzing data, and triggering recommendations.

[customer-screenshot]: images/customer.png
[product-screenshot]: images/product.png
[segment-screenshot]: images/segment.png
[transaction-screenshot]: images/transaction.png
[flow-screenshot]: images/flow.png
[biweekly-screenshot]: images/biweekly.png
[apiflow-screenshot]: images/apiflow.png