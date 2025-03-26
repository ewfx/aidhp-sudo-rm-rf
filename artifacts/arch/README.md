# Architecture : SUDO-RM-RF

## sudo-rm-rf Architecture

This documentation describes a personalized recommendation system for Wells Fargo products. The system is designed to recommend a product to a customer by analyzing his/her transactions, income, interests, and credit score while taking into account the product’s eligibility criteria. For demo purposes, the scheduler processes have not been implemented; instead, the recommendation service APIs (which the schedulers will eventually call) handle all calculations and database operations.

---

## 1. System Overview

The recommendation system is built to deliver personalized product suggestions for Wells Fargo customers. It operates by:

- **Analyzing Customer Transactions:** Reviewing a customer’s recent transactions (e.g., purchases, payments) to capture spending patterns and financial behavior.
- **Considering Customer Profile Data:** Using details such as annual income, credit score, and interests to understand the customer’s financial capacity and product fit.
- **Evaluating Product Eligibility:** Checking product eligibility criteria against the customer’s profile.
- **Generating Recommendations:** Constructing a context by combining the transaction history and the list of products that the customer is not currently using, and then invoking an LLM (via OpenAI Chat Completions) to recommend a product.

For demo purposes, although schedulers are envisioned to trigger these processes on a schedule, the current implementation exposes two APIs which perform all necessary validations, computations, and database calls.

---

## 2. Database Schema Context

### Data Collection

- **Product Data:** We scraped the Wells Fargo website to gather a comprehensive list of available products.
- **Customer Data:** A dataset of 30 customers was created. Customers are grouped into three segments: Individuals, Small Business, and Corporate. Based on the customer’s segment and income level, 1–2 products (representing what the customer is already using – **P1**) have been assigned to each customer profile.
- **Other Collections:** Additional collections include segments and transactions to support recommendation validations and context building.

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

Since the schedulers are not yet implemented, the system exposes two main APIs that will be called (by schedulers or manually) to trigger the recommendation process. These APIs encapsulate the logic for validating transactions and generating recommendations.

### API Endpoints

1. **Analyze Recommendable Transactions API**  
   **Endpoint:**  
   `POST /api/transactions/analyze_recommendable_transactions/by_date`  
   **Purpose:**

   - Accepts a date in the request body.
   - Iterates over all transactions that occurred on that day.
   - Applies validation rules (e.g., anomaly checks, funds verification) to mark each transaction as valid or invalid for recommendation.
   - Updates the transaction records in the database with a flag (`is_processed_for_recommendation`).

2. **Analyze Customer Product API**  
   **Endpoint:**  
   `GET /api/transactions/analyze_customer_product?customer_id=<customer_id>`  
   **Purpose:**
   - Accepts a `customer_id` as a query parameter.
   - Retrieves all transactions for the customer over a two-week period (for demo purposes, the current time is set to February 15, using transactions from Feb 1–15).
   - Fetches the customer profile which includes the list of products already used (**P1**).
   - Uses the customer’s segment ID to query the products collection and obtain all recommendable products (**P2**).
   - Computes the difference (**P2 - P1**) to identify products the customer is not currently using.
   - Constructs a prompt that combines the transaction summary and available products.
   - Calls the LLM via OpenAI Chat Completions to generate a recommendation.
   - Parses the JSON response (which contains the `transaction_id` used for context, the recommended product, and the reason for recommendation) and updates the recommendation results in the database.

### Flow Diagrams

#### A. Transaction Analysis API Flow

![Transaction Analysis][transaction-analysis-screenshot]

#### B. Customer Product Analysis API Sequence

![TransactProduction Analysis][product-analysis-screenshot]

---

## 4. Flask API Setup for Recommendation Service

In the Flask application, the controller endpoints correspond to the above APIs. The controller receives requests and then calls the service layer to:

- **For Transaction Analysis:** Validate transactions for a specified date and update the DB.
- **For Customer Product Analysis:** Retrieve customer data and transactions, compute the difference between currently used products (P1) and available products (P2), construct a prompt, call the LLM, and update the recommendation results accordingly.

The design separates concerns by delegating data retrieval, business logic, and LLM interaction to dedicated modules. This modularity allows for easier extension—such as integrating additional LLM models or refining recommendation rules—without impacting the overall system.

---

## 5. Additional Considerations

- **Error Handling & Logging:**  
  Both APIs incorporate logging to record key actions and errors (e.g., non-JSON LLM responses or missing transactions).

- **Scalability & Asynchronous Processing:**  
  In a production environment, consider processing transactions in batches and using asynchronous task queues (e.g., Celery) to handle high volumes.

- **Security & Configurations:**  
  Use environment variables for sensitive data (API keys, database credentials) and adhere to Flask security best practices.

- **Extensibility:**  
  The architecture’s layered design allows independent extension of controllers, services, and the recommendation engine.

---

## Conclusion

This design leverages API-driven endpoints to perform all necessary validations, computations, and LLM calls for personalized product recommendations. While the future schedulers will simply invoke these APIs with the appropriate parameters, the current implementation focuses on:

- **Transaction Validation:** Determining the recommendability of each transaction on a given day.
- **Customer Product Analysis:** Constructing a recommendation based on a two-week transaction summary and the difference between available products and those already in use.

# Production Readiness

The recommendation system can be built as an independent service delivering personalized product recommendations to various Wells Fargo Lines Of Business (LOBs). External applications onboard by providing their application metadata (app_info) and customer data. The customer records in the recommendation system are associated with an app_info identifier, linking them to the source application.

**Key Points:**

- **Application Onboarding:**  
  External applications supply their **app_info** (including a unique ID and the Kafka topic they subscribe to) along with customer data. The customer records maintain a reference to the corresponding app_info, ensuring clear traceability.

- **Data Ingestion via Kafka:**  
  When an application receives a transaction, it pushes the transaction data to a designated Kafka topic. The recommendation system, subscribed to these topics, ingests and stores the transaction in its database.

- **API-Driven Processing:**  
  The scheduler now acts as a trigger by calling dedicated recommendation APIs. These APIs are responsible for:

  - Validating and processing transactions received during the day.
  - Analyzing customer transaction history, computing product differences, constructing prompts, and invoking the LLM.
  - Updating database records with the final recommendation results.

- **Notification via Kafka:**  
  Once a recommendation is generated, the system retrieves the relevant app_info for the customer and pushes a notification containing the recommendation details to the Kafka topic provided by the external application.

---

### Visualizations

#### 1. Application Onboarding & Data Ingestion Flow

This flowchart illustrates how external applications onboard and how transaction data is ingested via Kafka.

![Application Onboarding][app-onboarding-screenshot]

#### 2. Scheduler API Call & Recommendation Notification Sequence

In this sequence diagram, the scheduler merely calls the recommendation system’s APIs. The APIs perform all the complex tasks—from processing transactions to generating recommendations and notifying the appropriate application via Kafka.

![Recommendation Sequence][recommendation-sequence-screenshot]

---

### Benefits for Line Of Business (LOB) Teams

- **Decoupled and Modular Design:**  
  The recommendation system operates independently of the scheduling mechanism. Schedulers simply trigger API calls, while the recommendation service manages all complex tasks, ensuring easier maintenance and scalability.

- **Centralized Recommendation Engine:**  
  The service delivers consistent, data-driven recommendations across multiple LOBs, ensuring that each application receives tailored insights based on customer transactions, income, credit score, interests, and product eligibility.

- **Real-Time Communication:**  
  By leveraging Kafka for both transaction ingestion and recommendation notifications, the system enables near real-time integration with external applications, allowing LOB teams to react promptly to new insights.

- **Efficient Resource Utilization:**  
  With the scheduler offloading computationally intensive tasks to the dedicated recommendation APIs, resources are optimized, and the system can scale to meet increasing data volumes without impacting scheduling performance.

---

This production readiness approach positions the recommendation system as a robust, scalable, and independent service that seamlessly integrates with various Wells Fargo LOBs. The clear separation between scheduling (triggering API calls) and processing (handling complex logic) ensures that the system can evolve and adapt to future business requirements while consistently delivering high-quality recommendations.

[customer-screenshot]: images/Customer.png
[product-screenshot]: images/product.png
[segment-screenshot]: images/segment.png
[transaction-screenshot]: images/transaction.png
[transaction-analysis-screenshot]: images/transaction-analysis.png
[product-analysis-screenshot]: images/product-analysis.png
[app-onboarding-screenshot]: images/app-onboarding.png
[recommendation-sequence-screenshot]: images/recommendation-sequence.png
[flow-screenshot]: images/flow.png
[biweekly-screenshot]: images/biweekly.png
[apiflow-screenshot]: images/apiflow.png
