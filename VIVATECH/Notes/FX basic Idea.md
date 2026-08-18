**Foreign Exchange (FX)** simply means:

> Converting an amount from one currency into another currency using an exchange rate.

For example:

`100 USD → XAF`

Suppose:

`1 USD = 600 XAF`

Then:

`100 USD × 600 = 60,000 XAF`

That's the basic idea.

But a production FX engine is much more than: `amount * rate`

You need to answer:
- Where does the rate come from?
- Is the rate real-time?
- Who provides it?
- How long is it valid?
- Which currency is the base?
- What happens if the provider is down?
- Do we add a markup?
- Do we charge an FX fee?
- Which rate was actually used for a transaction?
- How do we reproduce yesterday's transaction?
- What happens when USD → XAF is unavailable?
- Can we convert USD → EUR → XAF?
- How do we handle rounding?
- What happens if two requests arrive simultaneously?
- What happens if the exchange rate changes between quote and execution?

These are the things your FX design needs to handle.