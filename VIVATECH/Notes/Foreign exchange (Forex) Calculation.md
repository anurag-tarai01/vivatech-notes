Exchange rates are expressed as a pair: 
> **Base Currency / Quote (To) Currency** (e.g., `USD/EUR = 0.92`)

The value represents how many units of the quote currency are needed to purchase **1 unit** of the base currency.

---
## 1. Direct & Reverse Conversion Formulas
#### Direct Conversion (Base $\to$ Target)
When converting an amount from the **base** currency to the **target (quote)** currency, multiply by the exchange rate:
$$\text{Target Amount} = \text{Base Amount} \times \text{Exchange Rate}$$

**Example:** Converting $500\text{ USD}$ to $\text{EUR}$ where $\text{USD/EUR} = 0.92$:

$$\text{EUR Amount} = 500 \times 0.92 = 460\text{ EUR}$$
#### Reverse Conversion (Target $\to$ Base)
When converting backwards from the **quote** currency to the **base** currency, divide by the exchange rate (or multiply by the inverse rate $\frac{1}{\text{Rate}}$):

$$\text{Base Amount} = \frac{\text{Target Amount}}{\text{Exchange Rate}}$$

- **Example:** Converting $460\text{ EUR}$ back to $\text{USD}$ at $\text{USD/EUR} = 0.92$:

$$\text{USD Amount} = \frac{460}{0.92} = 500\text{ USD}$$
---
## 2. Cross-Rate Calculation (Indirect Pairs)

Most currency APIs provide rates pegged against a single anchor currency (typically $\text{USD}$). When converting between two non-anchor currencies (e.g., $\text{EUR} \to \text{JPY}$), compute the **cross-rate**:

$$\text{Rate}_{\text{Base} \to \text{Target}} = \frac{\text{Rate}_{\text{USD} \to \text{Target}}}{\text{Rate}_{\text{USD} \to \text{Base}}}$$
**Formula:**
$$\text{Target Amount} = \text{Amount} \times \left(\frac{\text{Rate}_{\text{USD} \to \text{Target}}}{\text{Rate}_{\text{USD} \to \text{Base}}}\right)$$

**Example:** Convert $1,000\text{ EUR}$ to $\text{JPY}$:

- $\text{USD/EUR} = 0.92$
    
- $\text{USD/JPY} = 155.00$
    
- $\text{EUR/JPY Rate} = \frac{155.00}{0.92} \approx 168.478$
    
- $\text{Converted Amount} = 1,000 \times 168.478 = 168,478\text{ JPY}$

---
## 3. Bid-Ask Spread Adjustments
Live market feeds provide two prices: the **Bid** (selling price) and the **Ask** (buying price):

| **Action**                | **Relevant Price** | **Formula**                                                     |
| ------------------------- | ------------------ | --------------------------------------------------------------- |
| **Selling Base Currency** | Bid Rate (Lower)   | $\text{Target Received} = \text{Base Amount} \times \text{Bid}$ |
| **Buying Base Currency**  | Ask Rate (Higher)  | $\text{Base Received} = \frac{\text{Target Spent}}{\text{Ask}}$ |
