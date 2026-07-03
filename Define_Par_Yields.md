# ☝ Term Structure ⇄ Par Yields

## 🤔💭 Defining Par Yield
The par yield is the exact coupon rate necessary for a bond's market price to equal its principal (or face) value.

## 🔣 The Par Yield Formula
We derive the par yield by starting with the standard present value equation for a bond. By setting the bond's total price equal to its principal, we get:

<br>


$$\text{Principal} = \sum_{t=1}^{T}\gamma_t\times \text{PV}(t)\times \text{Par Yield}\times\text{Principal}+ \text{PV}(T)\times\text{Principal}$$

<br>

By dividing by the $\text{Principal}$ and isolating the $\text{Par Yield}$, the formula simplifies to:

<br>

$$\text{Par Yield}=\frac{1-PV(T)}{\sum _{t=1}^{T}\gamma_t\times PV(t)}$$

<br>

In simple terms, the par yield is equal to one minus the present value factor at maturity ($\text{PV(T)}$) divided by the sum of the present value factors ($\gamma_t\times \text{PV(t)}$) for all payment dates. Remember that a present value factor is the price of a zero-coupon bond for a given maturity derived directly from prevailing spot rates.

## ✖️➗➕➖ Mechanics of the Par Yield Formula
Breaking down the mathematical components clarifies how par yields react to shifting interest rates and time horizons:

* **The Numerator $1−PV(T)$**: This is the difference between the face value of 1 and the present value of that 1 received at maturity ($T$). This value grows larger as maturity extends or as interest rates climb.

* **The Denominator ${\sum _{t=1}^{T}\gamma_t\times PV(t)}$**: This is the sum of all present value discount factors from today up to time $T$. While extending the maturity naturally adds more terms and increases this sum, higher interest rates reduce the value of those factors, causing the sum to shrink.

* **Low-Rate Boundary**: As interest rates approach zero, the numerator goes to zero, and the denominator approaches the total number of periods times the  frequency ($\gamma_i\times t_i$). All par yields go to zero.

* **High-Rate Boundary**: As interest rates approach infinity, the terminal present value factor vanishes (pushing the numerator to 1), and the sum of all discount factors collapses, driving the denominator to zero. All par yields go to infiinity.

* **The Core Takeaway**: Par yields serve as an elegant summary metric. They capture the aggregate level of the term structure from today to the maturity date.



## 💡🎯 Why Par Yields Matter

* **For Issuers**: It signals the precise coupon rate required to price new debt at par, driving the initial calculation of borrowing costs.

* **For Corporations**: It acts as a benchmark to evaluate the expense of their outstanding debt and informs their overall cost of capital.

* **For Investors**: It provides a baseline periodic return measure to assess opportunities within the fixed-income market.

## 🔍Illustrative Example:

Consider an example with a six-month and one-year annualized spot rate of 5%. The zero prices (or present value factors) are calculated as:

Six-month :
>$\text{PV}(0.5) = e^{-0.05\times 0.5} = \text{ 0.9753}$

One-year:
>$\ \ \text{  PV}(1.0) = e^{-0.05\times 1.0} = \text{ 0.9512}$

Applying the par yield formula gives us the semi-annual coupon rate:

>$Par Yield=0.5\times (0.9753+0.9512)1−0.9512​=5.07\%$

If we plug this 5.07% coupon rate back into the standard pricing equation for a bond with a 100 face value, it perfectly recovers the principal:


>$\$100=\$2.532\times 0.9753+\$102.532\times 0.9512$


## 🔢 Applying the Nelson-Siegel Model
Calculating an exact par yield requires a known term structure of interest rates. Because observable market data only provides discrete zero prices at specific intervals, we need a mathematical model to generate a continuous yield curve to price bonds at any arbitrary maturity.
In the previous volume, we solved this by estimating a Nelson-Siegel model using FEDInvest data across 399 specific securities. We will now take those theoretical estimates and apply them practically using a three-step approach:


* **Calculating Par Yields**: We will leverage estimtes of the Nelson-Siegel model to generate a continuous curve, allowing us to calculate custom par yields across various maturities.
* **Benchmarking Against FRED**: We will compare our internal calculations directly against the institutional benchmarks published by the Federal Reserve Economic Data (FRED) database.
* **Reverse-Engineering With FRED Data**: We will work backward from FRED's published par yields to extract the implicit Nelson-Siegel parameters driving their institutional estimates.

This progression bridges the gap between theoretical modeling and real-world fixed-income trading.


## Reminder‼️: The Nelson-Siegel Parameters
The Nelson-Siegel model is powerful because it constructs the entire yield curve using a single equation governed by four interconnected parameters. The spot rate $r(t)$ for any maturity t is expressed as:

<br>

$$r(t) = \beta_0 + \beta_1 \left( \frac{1 - e^{-t/\tau}}{t/\tau} \right) + \beta_2 \left( \frac{1 - e^{-t/\tau}}{t/\tau} - e^{-t/\tau} \right)$$

<br>

Each parameter governs a specific shape characteristic of the term structure:

* **Level ($\beta_0$​)**: Describes the long-term yield. This is the steady-state baseline that the yield curve approaches as the time to maturity extends toward infinity. If the overall level of interest rates rises or falls, this parameter shifts the entire curve up or down.

* **Slope ($\beta_1$)**: Describes the short-term yield component and dictates the steepness of the curve. A negative value typically creates a normal, upward-sloping yield curve, while a positive value flattens or inverts the curve. Keep in mind that $r(0)$ (the instantaneous short rate) equals $\beta_0+\beta_1$.

* **Curvature ($\beta_2$)**: Describes the medium-term yields. It dictates the magnitude and direction of the "hump" (or U-shape) often observed in the middle maturities before the curve flattens out.

* **Decay Factor ($\tau$)**: Acts as the shaping parameter. It controls how quickly the short-term slope decays into the long-term level, which ultimately determines the exact maturity at which the curvature component ($\beta_2$) reaches its maximum impact.

By estimating these four parameters (as we previously did using 399 securities from FEDInvest), we can generate a continuous, smooth yield curve that perfectly sets up the calculation of par yields for any desired maturity.

Comparing our results to FRED and reverse engineering estimates demonstrates the importance of data.  What we call the Horse and Rabbit stew.


># [🐴🐰🍲 ](https://youtu.be/hi68J8T6Ma4)
