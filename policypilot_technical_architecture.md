# Technical Architecture: PolicyPilot (Booking & German Tax Automation)

This document details how **PolicyPilot** can programmatically handle flight/hotel booking and automate German travel tax compliance without the overhead of becoming an official travel agency.

---

## 1. High-Level Architecture Overview

PolicyPilot operates as a lightweight **AI Orchestration Overlay**. Instead of building booking inventories or payment clearing houses, it leverages specialized APIs for travel fulfillment, OCR extraction, and accounting sync.

```mermaid
graph TD
    User([Employee in Slack/Teams/Web]) -->|Chat: "Book Munich trip"| AI[PolicyPilot AI Engine]
    AI -->|Fetch flights & hotels| TravelAPI[TravelPerk / Duffel API]
    TravelAPI -->|Return options| AI
    AI -->|Filter options based on company policy| User
    User -->|Confirm & Approve| AI
    AI -->|Trigger Booking| TravelAPI
    TravelAPI -->|Send structured Invoice/Receipt JSON| TaxEngine[PolicyPilot Tax & Per-Diem Engine]
    TaxEngine -->|Extract VAT splits: 7% vs 19%| OCR[AI OCR Receipt Parser]
    TaxEngine -->|Calculate BMF Per-Diem deductions| TaxDB[(BMF Yearly Rates)]
    TaxEngine -->|Export DATEV format| DATEV[DATEV Accounting System]
```

---

## 2. Part 1: Handling Hotel & Flight Booking (Fulfillment)

To avoid merchant-of-record liability, IATA registration, and 24/7 travel support infrastructure, PolicyPilot can integrate with two API routes:

### Route A: The Partner-First API (Recommended for B2B)
*   **Provider:** **TravelPerk API** or **Navan API**
*   **Mechanism:** Your startup connects to a corporate customer's existing TravelPerk account via OAuth 2.0. 
*   **Workflow:**
    1.  User chats with PolicyPilot: *"I need a flight to Munich next Tuesday."*
    2.  PolicyPilot queries `GET /v2/trips` and searches flights/hotels using TravelPerk's marketplace.
    3.  PolicyPilot parses the results, applies the company's internal travel policy rules (e.g., *"Flights under €200 only, hotels max €150/night"*), and highlights compliant choices.
    4.  Once the user selects a choice, PolicyPilot hits `POST /v2/bookings` to purchase the ticket.
*   **Why it works:** TravelPerk handles credit card billing, consolidated monthly invoicing, changes/cancellations, and customer support. You only provide the smart AI interface and policy auditing.

### Route B: The Infrastructure Aggregator API (If building custom travel apps)
*   **Provider:** **Duffel API** (Flights) and **Amadeus Self-Service APIs** (Hotels)
*   **Mechanism:** Duffel connects directly to airline NDC (New Distribution Capability) portals.
*   **Workflow:**
    1.  PolicyPilot handles the search directly via Duffel's REST API.
    2.  PolicyPilot acts as the booking agent, charging the user's corporate credit card programmatically.
*   **Why it works:** Gives you total control over the booking interface, margins, and custom pricing markups. However, you must handle booking errors and travel support.

---

## 3. Part 2: Handling German Taxes (*Reisekosten*)

German travel tax compliance is highly administrative, making it an ideal target for automation.

### A. Per-Diem (*Verpflegungsmehraufwand*) Calculations
Germany requires calculating tax-free food allowance based on time spent away from the home/work base:
*   **Daily Flat Rates:** Currently €30 for a full 24-hour day, and €14 for arrival/departure days (domestic Germany). These values are updated yearly by the Federal Ministry of Finance (*Bundesministerium der Finanzen* - BMF).
*   **The Meal Deduction Rule:**
    *   If a hotel invoice includes **breakfast** (or it is included in the room rate), you must deduct exactly **20% of the full day's per-diem** (currently €6.00 deduction).
    *   If the company or a client provides **lunch or dinner**, you must deduct exactly **40% of the full day's per-diem** (currently €12.00 deduction per meal).
*   **PolicyPilot Automation:**
    ```python
    def calculate_german_per_diem(trip_itinerary, breakfast_included=False, lunch_provided=False):
        # 1. Base rate calculation based on hours traveled
        daily_rate = 30.00 if trip_itinerary.is_full_day else 14.00
        
        # 2. Apply BMF standard deductions (based on the full day rate of 30.00)
        deductions = 0.00
        if breakfast_included:
            deductions += 30.00 * 0.20  # €6.00 deduction
        if lunch_provided:
            deductions += 30.00 * 0.40  # €12.00 deduction
            
        final_per_diem = max(0.00, daily_rate - deductions)
        return final_per_diem
    ```

### B. Automated VAT (*Umsatzsteuer*) Splits
German receipts split taxes across different rates for hotel bookings:
*   **7% VAT:** Applies strictly to the overnight lodging accommodation (*Logis*).
*   **19% VAT:** Applies to all auxiliary services, including breakfast (*Frühstück* / *Business Package*), parking, Wi-Fi, and mini-bar items.
*   **PolicyPilot Automation:**
    1.  When TravelPerk or Duffel generates the billing PDF, PolicyPilot runs it through a document parser (e.g., AWS Textract or Azure Document Intelligence).
    2.  An LLM prompts parses the text: *"Extract all line items, identifying their net amount, VAT rate (7% vs 19%), and tax amount."*
    3.  It automatically maps the overnight stay to the 7% tax account and the breakfast/business package to the 19% tax account.

---

## 4. Part 3: Financial Exports (DATEV Integration)

For a German SME to adopt PolicyPilot, the output must be compatible with their accountant's workflows.

*   **DATEV Import Format:** You must export expenses as a standardized CSV file matching the **DATEV Buchungsstapel** structure.
*   **Mapping Table:**
    *   **Lodging Net (7% VAT):** Maps to DATEV account code **4650** (Bewirtungskosten / Reisenebenkosten) or specific corporate accounts.
    *   **Per-Diems (Tax-Free):** Maps to DATEV account code **4653** (Fahrtkosten / Kilometerpauschale) or **4664** (Reisekosten Arbeitnehmer).
*   **DATEV XML Interface (Online):** For a more premium experience, connect to **DATEV Unternehmen online** via their official OAuth REST API, pushing the receipt image and the booking data directly to their cloud accounting inbox.

---

## 5. Lean Tech Stack for the MVP (Fast Launch)

As an experienced software engineer, you can launch the PolicyPilot MVP in under 4 weeks with this lightweight stack:

1.  **AI Engine:** FastAPI (Python) or NestJS (TypeScript) utilizing **LangChain** or **LlamaIndex** to process booking intents and parse invoices.
2.  **Travel Fulfillment:** Duffel API client (Python/TS SDK) or TravelPerk Developer Sandbox.
3.  **UI Front-End:** Slack Bolt API / Microsoft Teams SDK (building interactive cards using Block Kit / Adaptive Cards). This completely avoids having to design a complex web UI dashboard for the initial launch.
4.  **Database:** PostgreSQL for storing policy rules, user profiles, and tax audit trails.
